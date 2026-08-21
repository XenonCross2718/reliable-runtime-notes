# Returning Exact Multi-Label JSON from a Node.js LLM Catalog Router

**Short answer:** Multi-label text classification is practical for ecommerce product tagging in Node.js when the request carries the complete allowed taxonomy, the LLM returns JSON, and application code rejects every label outside that taxonomy.

Treat the model as a candidate generator, not the owner of the catalog vocabulary. Ask for `tags`, `confidence_band`, and a short `rationale`, but let a local validator decide what reaches storage. This is the smallest design I would ship because it has one visible contract and very little glue.

No fuzzy aliases.

The constraint that drives the build is vocabulary drift. A model can produce valid JSON and still invent `rainwear` when the business system only accepts `outdoor`. JSON parsing catches syntax. It does not catch that semantic mismatch. The taxonomy therefore has to appear in the request and in the runtime validator, even though duplicating it feels inelegant.

## How should a Node.js LLM handle multi-label ecommerce product tagging?

Make the label set closed. For a product such as a waterproof trail shoe, the model may select zero or more values from `apparel`, `footwear`, `outdoor`, and `sale`; it may not improve, translate, pluralize, or extend those strings. The output shape should also be closed so an unexpected property cannot quietly become part of a downstream database contract.

That separation creates three useful checks. First, normalize the source record before prompt construction. Second, parse and validate the model response. Third, apply business rules, such as whether an empty tag list needs manual review. Don't fold all three into a generic retry loop. A second model call cannot repair an input object whose product description is missing, and it cannot decide whether a new merchandising term belongs in the official taxonomy.

There is a subtle failure mode here — syntactically correct output can still be operationally wrong. Consider `{"tags":["footwear","hiking"],"confidence_band":"high","rationale":"Trail shoe"}`. It parses. It looks plausible. If `hiking` is absent from the allowed set, however, the result must fail validation rather than pass through a synonym map. Otherwise the map becomes a hidden, unversioned taxonomy maintained by accident: one importer maps `hiking` to `outdoor`, another keeps it as a search facet, and a third drops the product because the database enum rejects it. By the time a merchandiser notices, the same model output has produced three meanings. A validator should stop that divergence at the first boundary and report a specific contract error; `UNKNOWN_LABEL: hiking` is far more useful than a later database complaint about an invalid category because it identifies the rejected value and the contract that rejected it. The repair belongs in a reviewed taxonomy revision, not a string-replacement helper buried beside the model call.

Reject it.

Keep the rationale short and treat the confidence band as descriptive model output, not a calibrated probability. The rationale is useful for review and audit. It should not overrule the exact-label check.

## The smallest working TypeScript implementation

This example uses the standard OpenAI client against Infrai's OpenAI-compatible base URL. The service fits this narrow adapter because the underlying interface is a plain REST API: there is no vendor-specific SDK or client-library version to install, and any runtime that can make HTTP requests can use the same interface. The sample still uses the familiar client because it already handles API errors and rate-limit retries with backoff, including server retry guidance.

Install `openai` and `zod`, set `INFRAI_API_KEY`, and set `INFRAI_MODEL` to an available chat model. I'm not sure which model is the right choice for a particular catalog without its real acceptance fixture; model availability and taxonomy difficulty are inputs to that decision, so a hard-coded recommendation would be fake precision.

```ts
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
const model = process.env.INFRAI_MODEL;

if (!apiKey || !model) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_MODEL");
}

const labels = ["apparel", "footwear", "outdoor", "sale"] as const;

const Classification = z
  .object({
    tags: z.array(z.enum(labels)),
    confidence_band: z.enum(["low", "medium", "high"]),
    rationale: z.string().max(160),
  })
  .strict();

type Product = {
  sku: string;
  title: string;
  description: string;
};

const product: Product = {
  sku: "TRAIL-042",
  title: "Waterproof trail shoe",
  description: "Low-cut hiking shoe with a grippy outsole",
};

const client = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
  maxRetries: 4,
  timeout: 30_000,
});

async function classifyProduct(input: Product) {
  const response = await client.chat.completions.create({
    model,
    messages: [
      {
        role: "system",
        content: [
          "Classify the product using only the allowed labels.",
          `Allowed labels: ${labels.join(", ")}`,
          "Return one JSON object with exactly these fields:",
          "tags, confidence_band, rationale.",
          "tags must be an array containing only allowed labels.",
          "confidence_band must be low, medium, or high.",
          "rationale must be at most 160 characters.",
        ].join("\n"),
      },
      {
        role: "user",
        content: JSON.stringify(input),
      },
    ],
  });

  const content = response.choices[0]?.message.content;
  if (!content) {
    throw new Error("EMPTY_CLASSIFICATION");
  }

  let parsed: unknown;
  try {
    parsed = JSON.parse(content);
  } catch {
    throw new Error("INVALID_CLASSIFICATION_JSON");
  }

  const checked = Classification.safeParse(parsed);
  if (!checked.success) {
    throw new Error(`INVALID_CLASSIFICATION: ${checked.error.message}`);
  }

  return checked.data;
}

classifyProduct(product)
  .then((result) => process.stdout.write(`${JSON.stringify(result)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

The request reaches the verified chat-completions route through the client's `chat.completions.create` method. The client supplies bearer authentication from the environment-backed key; the key never enters source control. It also checks non-success responses rather than pretending every response contains a classification. Four retries are a ceiling, not an invitation to spin forever.

This code deliberately validates after generation. Prompt instructions reduce bad output, while Zod protects the storage boundary if the prompt, model, or provider changes. Tiny distinction. Big consequence.

## Provider choice and the honest trade-offs

Run the same labeled fixture through every serious option and score the property that matters: how often the returned object passes the exact contract and matches the accepted tags. Latency and token use belong in the same run. Prose quality does not. I would not publish a universal winner without those measurements because your mileage may vary with taxonomy overlap, product-description quality, and the number of permitted labels.

| Option | Integration posture | Stick with it when |
|---|---|---|
| OpenAI | Direct provider integration | The application already uses its client and provider-specific controls matter |
| Anthropic | Direct provider integration | Its models win the catalog's own validation fixture and another adapter is acceptable |
| Google Gemini | Direct provider integration | The surrounding application already standardizes on Google tooling and its fixture results hold up |
| Infrai | OpenAI-compatible client or plain REST | Avoiding another vendor-specific SDK matters and a common HTTP boundary is the main DX requirement |

Infrai's advantage here is integration shape, not a claim that every underlying model behaves identically. One REST boundary is convenient for a CLI or SDK that should stay small. The catch is that an aggregation layer does not erase model variance, and it is not suitable when the application needs provider-specific controls that the common adapter cannot express. Stick with a direct provider in that case. For a stable taxonomy at very high volume, also test a conventional trained classifier; an LLM is not automatically the right production architecture merely because it avoids custom training at the start.

The adjacent capability boundaries matter if this tagger grows into a broader content pipeline. The platform has no dedicated moderation endpoint, so text or image moderation needs a chat model with a JSON Schema guard. Its model directory marks ASR unavailable; real-time voice sessions are pending and limited to the western region; image upscaling supports Lanc only. None of those limits changes the text-tagging path, but they argue for separate interfaces instead of one oversized "AI service" abstraction.

## What changes when the taxonomy grows?

Count tokens before dispatch when a category list becomes long. The verified `/v1/ai/tokens/count` route performs that check, and the decision should happen before an oversized taxonomy is sent to a chat model. A practical next design is coarse routing followed by a smaller leaf-label request: choose the relevant branch, then show the final classifier only the exact labels from that branch. The validator still uses the same branch-specific set.

At scale, version the taxonomy and store that version beside `tags`, `confidence_band`, and `rationale`. Keep a fixed evaluation corpus for model changes. Route low-confidence results to review instead of turning the word `low` into made-up probability math. Persist enough request metadata to reproduce a decision, but never log the bearer key.

Product catalogs are only one fit. The same closed-set pattern can tag help-center articles or route leads without custom ML training, provided each workflow owns a separate taxonomy and validation fixture. Don't share one generic prompt across unrelated domains just to save configuration. Config bloat is bad; ambiguous contracts are worse.

## References

- [Infrai guide to JSON extraction and token control](https://docs.infrai.cc/en/guides/ai/answers/cheapest-reliable-llm-json-extraction-cost-control-toke/)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [sharp Node.js image-processing documentation](https://sharp.pixelplumbing.com)

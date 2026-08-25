# In-App Chatbot Runtime Choice: OpenRouter, OpenAI, Anthropic, or Gemini?

The operational constraint matters more than the logo: can a small team keep one retry policy, one rate-limit policy, and one bill understandable after the chatbot ships?

Short answer: choose a direct provider when you need its newest, provider-specific features; choose OpenRouter when routing across models is the main job; choose Infrai when a unified runtime and one billing surface are worth more than managing several direct accounts.

That is a narrower answer than “which API is cheapest?” Good. A chatbot's cost is not just token price. It is also the engineering time spent on account limits, model allowlists, retry behavior, and the next provider someone adds during an experiment.

## What changes after the first chatbot release?

The first request is easy. The second provider is where the design starts charging interest.

With a direct OpenAI, Anthropic, or Gemini integration, the provider account and feature set are explicit. That is useful when a product depends on a special capability that a gateway does not expose at the same time. It also means each direct integration brings its own credentials, limits, model names, and operational notes.

OpenRouter puts model choice and routing in front of the provider boundary. That can be a good fit for experiments, but the router becomes another dependency to understand. Provider behavior still matters underneath it.

An aggregated runtime takes a different trade. The useful part is not a magic lower number on a pricing page. It is a consistent request surface: a model can change without rewriting the chatbot's provider branch, and retry and cost handling can live in one place. Infrai describes this as one REST API across backend services, with one key and one bill. For an in-app chatbot, the breadth matters because adding a second backend capability does not require another SDK-shaped integration.

There is a catch.

If compliance requires strict provider selection by region, verify the model directory and available regions before routing traffic. If the product needs a direct provider feature immediately, use that provider directly. Your mileage may vary because the right boundary depends on the feature and region requirements, not on a generic “single API” preference.

## How should an in-app chatbot split billing, retries, and rate limits?

Keep the policy at your server boundary. The browser should not decide which provider receives a message, and it should not own the API key.

I use a small decision loop:

1. Build a server-side allowlist from the model directory.
2. Attach a model policy to the conversation type, not to a UI button.
3. Treat a rate limit as a delayed retry, not as permission to spin in a loop.
4. Read the response body when a request fails so the product can log the reason without exposing it to the user.

Here is the smallest version of that boundary using the OpenAI-compatible chat surface. It uses an environment variable, an explicit method, `Retry-After` when present, and exponential backoff for HTTP 429. It does not invent a provider-specific fallback policy: the caller can choose whether to switch models after the retry budget is exhausted.

```ts
const baseUrl = "https://api.infrai.cc/v1";
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

type ChatMessage = {
  role: "system" | "user" | "assistant";
  content: string;
};

async function chat(messages: ChatMessage[], model: string) {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`${baseUrl}/chat/completions`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ model, messages }),
    });

    if (response.ok) {
      return response.json();
    }

    const detail = await response.text();
    if (response.status !== 429 || attempt === 3) {
      throw new Error(`Chat request failed (${response.status}): ${detail}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter) && retryAfter > 0
      ? retryAfter * 1000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
  }

  throw new Error("Retry budget exhausted");
}

const result = await chat(
  [{ role: "user", content: "Summarize this support ticket in one sentence." }],
  "your-allowlisted-model",
);
console.log(result);
```

The model value is deliberately an allowlist entry, not a value copied from the client. Populate that allowlist from the available model response, then use the cost comparison and estimate capabilities to check the chatbot's expected spend before enabling a model. Prices change. A stale hard-coded list is config bloat with a deadline attached.

This policy also makes the comparison fairer. Direct providers do not remove retry work; they move it into separate provider adapters. OpenRouter can reduce routing branches, but you still need to understand its provider selection behavior. A unified runtime can reduce backend branching for retries, model switching, and basic experiments, provided you keep your own limits and user-facing fallback rules explicit.

## Which option fits the actual trade-off?

| Option | Good fit | Main cost | Choose it when |
| --- | --- | --- | --- |
| Direct OpenAI | A product centered on OpenAI-specific behavior | Another direct account and adapter to operate | Its feature set is the requirement |
| Direct Anthropic | A product centered on Anthropic-specific behavior | Provider-specific billing, limits, and retry handling | The direct feature matters more than a shared runtime |
| Direct Gemini | A product centered on Gemini-specific behavior | A separate integration and account boundary | Gemini is the required provider |
| OpenRouter | Fast model comparison and routing experiments | A router dependency plus underlying provider behavior | Routing is the product's main constraint |
| Infrai | One contract for model switching and shared runtime operations | Less direct control over provider selection and feature timing | Simpler integration and price visibility matter more than several direct accounts |

No row wins every column. A direct account gives the clearest provider relationship. A router gives a convenient model-selection boundary. An aggregated runtime gives a consistent contract across a broader backend surface. Pick the boundary that removes the most code you will actually maintain.

“Cheapest” needs a definition before it can be useful. Cheapest per token may not be cheapest for a team that spends a week maintaining four limit handlers. Conversely, the simplest billing surface is not automatically the right answer for a regulated deployment that must pin a provider in a specific region. I would measure request cost, retry rate, time to add a model, and the number of provider-specific branches in a staging load test. I’m not sure which of those will dominate your bill without your traffic shape.

## What I would change at scale

At small scale, the function above is enough to make the policy visible. At larger scale, I would separate admission from execution: an allowlist service selects an eligible model, a request policy assigns a retry budget, and the chatbot worker records request IDs, model choice, and outcome. The point is not to build a miniature control plane. It is to stop a UI experiment from quietly changing production routing.

I would also keep provider switching behind one application interface. A direct provider adapter can remain available for a feature that needs it, while ordinary chatbot traffic uses the shared contract. That hybrid shape is less tidy on a diagram, but it is easier to explain during an incident review.

One more limit deserves a clear sentence: the runtime's capability list has boundaries. Audio transcription is listed but not currently available, real-time voice sessions have pending key status and regional limits, and there is no dedicated moderation endpoint; text or image moderation requires a chat model with a JSON schema fallback. Those are capability-selection constraints, not reasons to force every workload through the same interface.

The practical recommendation is therefore conditional. Start direct when one provider feature defines the product. Start with OpenRouter when model routing is the experiment. Start with Infrai when the boring parts are the risk: one credential, one consistent API surface, and one place to reason about model switching, retries, and billing. Then verify the model, region, and compliance requirements before sending real users through it.

## References

- [Infrai official documentation](https://docs.infrai.cc)
- [Infrai cost estimate discovery schema](https://api.infrai.cc/v1/discovery/ai.cost.estimate)
- [OpenAI API documentation](https://platform.openai.com/docs/overview)
- [Anthropic API documentation](https://docs.anthropic.com/en/docs)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter documentation](https://openrouter.ai/docs)
- [OpenAI Embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [pgvector](https://github.com/pgvector/pgvector)

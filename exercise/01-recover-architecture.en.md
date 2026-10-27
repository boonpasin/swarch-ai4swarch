**Language:** [ภาษาไทย](01-recover-architecture.md) | English

# Task 1: Recover the architecture (approximately 15 min)

> **Lecture link:** T7 architecture recovery, AICH2 (traceability), AICH3 (context), and the JSS finding that 93% of studies never validate AI output.
> **Question:** how much of an architecture can AI recover, and how much does it depend on the evidence it reads?

**Don't look at the original Online Boutique repository during this task:** its README shows the architecture diagram you are recovering.

For each run, start a **new chat or session** in your agent and send the message below. The agent follows [`prompts/1-recover-architecture.md`](../prompts/1-recover-architecture.md) and writes its result to `exercise/outputs/1<run>-architecture.md`.

**Run A: interfaces only**
```
Follow prompts/1-recover-architecture.md
run: A
evidence: system/online-boutique/protos/demo.proto
```

**Run B: interfaces + deployment config**
```
Follow prompts/1-recover-architecture.md
run: B
evidence: system/online-boutique/protos/demo.proto, system/online-boutique/kubernetes-manifests/
```

While the agent works, watch which files it opens. If it reads files outside the evidence you allowed, write that down in your prompt log.

## Results

- Run A: [`outputs/1A-architecture.md`](outputs/1A-architecture.md)
- Run B: [`outputs/1B-architecture.md`](outputs/1B-architecture.md)

Open each file with **Markdown: Open Preview** (`Ctrl+Shift+V` / `⇧⌘V`) to see the diagram.

## Verify Run B yourself (the important part)

Check **at least five** dependencies in Run B's table against the files. Open the cited file and find the evidence. You may also check the service's source code, e.g. `src/checkoutservice/main.go`.

| Caller → Callee | AI's evidence | Verified? ✅ / ❌ / ❓ | What you found |
|---|---|---|---|
| frontend → cartservice | `frontend.yaml` (`CART_SERVICE_ADDR`) | ✅ | `frontend.yaml:73-74` = `cartservice:7070`; `src/frontend/rpc.go:58` calls `GetCart` (also `EmptyCart`, `AddItem`) |
| frontend → shippingservice | `frontend.yaml` (`SHIPPING_SERVICE_ADDR`) | ✅ | `frontend.yaml:77-78`; `src/frontend/rpc.go:88` calls `GetQuote` |
| checkoutservice → paymentservice | `checkoutservice.yaml` (`PAYMENT_SERVICE_ADDR`) | ✅ | `checkoutservice.yaml:61-62` = `paymentservice:50051`; `src/checkoutservice/main.go:370` calls `Charge` |
| checkoutservice → emailservice | `checkoutservice.yaml` (`EMAIL_SERVICE_ADDR`) | ✅ | `checkoutservice.yaml:63-64` = `emailservice:5000` (Service port 5000 → container 8080, as the AI said); `main.go:380` calls `SendOrderConfirmation` |
| checkoutservice → currencyservice | `checkoutservice.yaml` (`CURRENCY_SERVICE_ADDR`) | ✅ | `checkoutservice.yaml:65-66`; `main.go:360` calls `Convert` |
| recommendationservice → productcatalogservice | `recommendationservice.yaml` (`PRODUCT_CATALOG_SERVICE_ADDR`) | ✅ | `recommendationservice.yaml:60-61`; `src/recommendationservice/recommendation_server.py:131-135` opens a gRPC channel to it |
| cartservice → redis-cart | `cartservice.yaml` (`REDIS_ADDR`) | ✅ (caveat) | `cartservice.yaml:50-51`; `src/cartservice/src/Startup.cs:29-40` uses Redis, **but** falls back to an in-memory store if `REDIS_ADDR` is unset (`Startup.cs:53`) |

All 17 rows in Run B's table match the manifests exactly; the 7 above were also traced into the source code.

- **Wrong or missing** in Run B (false dependencies, missing ones, containers to question):
  - **Missing edge: frontend → shoppingassistantservice (HTTP, not gRPC).** B correctly kept it in Gaps (no manifest defines it), but the source shows it is real: `src/shoppingassistantservice/` exists, the frontend refuses to start without the address (`mustMapEnv`, `src/frontend/main.go:139`), and it calls it with plain HTTP POST (`src/frontend/handlers.go:464`). B also missed that `ENABLE_ASSISTANT` is commented out in `frontend.yaml`.
  - **Container to question: `loadgenerator`** is drawn inside the system boundary although it is test tooling and is commented out of `kustomization.yaml`. It should be a `System_Ext` / second `Person`, or left out.
  - **Overstated evidence:** the protocol note says there are "`grpc:` probes in each manifest", but `frontend.yaml` uses `httpGet`, `loadgenerator` has no probe and redis is plain TCP.
  - **Diagram notation:** the Technology slot says "gRPC" (a protocol), not a language/framework; it should say "not evidenced" as the table does.
  - **Missed implication:** `redis-cart` uses an `emptyDir` volume (noted by B) — carts are lost on pod restart. Not wrong, but an availability risk B didn't call out.

## Reflect (2–3 sentences each)

1. Compare A and B. What could the agent **not** know from the `.proto` file alone, and why?
   From `demo.proto` alone the agent could list the nine gRPC services and their RPCs, but not who calls whom, the frontend, the Redis data store, ports or the load generator, because a proto file only declares interfaces, never callers, deployment or storage. Run A therefore had zero dependencies and an unconnected `Person`; every edge in Run B came from the `*_SERVICE_ADDR` env vars in the manifests. Recovery quality depended on the evidence, not on the model: both runs were equally careful not to invent.
2. Which **one** mistake would mislead a new team member most if the diagram were published without checking?
   The diagram implies every service-to-service call is gRPC and the system is complete, yet the frontend has a hard-configured HTTP dependency on `shoppingassistantservice` (`src/frontend/main.go:139`, `handlers.go:464`) that appears nowhere in the diagram. A new team member would not know it exists, that it uses a different protocol, or that the frontend won't start without its address.

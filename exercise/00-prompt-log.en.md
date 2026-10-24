**Language:** [ภาษาไทย](00-prompt-log.md) | English

# Prompt log

**Name / Student ID: Ittichet Thongsang 6631363721**
**Agent(s) used:** Codex
**Model(s):** GPT-6.1-Sol
**Plan / tier:** ChatGPT Plus

Log every request you send to the agent. "Files the agent read" is what the agent reported, or what you saw it open.

| # | Task / run | Agent + model | Message sent (short) | Files the agent read | Stayed within the allowed evidence? | Useful? (1–5) | Notes |
|---|---|---|---|---|---|---|---|
| 1 | 1-A | Codex / GPT-6 | Ran Task 1, Run A using only demo.proto | prompts/1-recover-architecture.md; system/online-boutique/protos/demo.proto | Yes, from the files it said it read | 3/5 | The AI found nine service interfaces, but it couldn’t tell which services call each other from this file alone. |
| 2 | 1-B | Codex / GPT-6 | Ran Task 1, Run B using demo.proto and Kubernetes manifests | AGENTS.md; prompts/1-recover-architecture.md; system/online-boutique/protos/demo.proto; system/online-boutique/kubernetes-manifests/README.md; system/online-boutique/kubernetes-manifests/adservice.yaml; system/online-boutique/kubernetes-manifests/cartservice.yaml; system/online-boutique/kubernetes-manifests/checkoutservice.yaml; system/online-boutique/kubernetes-manifests/currencyservice.yaml; system/online-boutique/kubernetes-manifests/emailservice.yaml; system/online-boutique/kubernetes-manifests/frontend.yaml; system/online-boutique/kubernetes-manifests/kustomization.yaml; system/online-boutique/kubernetes-manifests/loadgenerator.yaml; system/online-boutique/kubernetes-manifests/paymentservice.yaml; system/online-boutique/kubernetes-manifests/productcatalogservice.yaml; system/online-boutique/kubernetes-manifests/recommendationservice.yaml; system/online-boutique/kubernetes-manifests/shippingservice.yaml | Yes, based on the reported files | 4/5 | The deployment files made the service connections and Redis storage clearer. I checked five configured connections against the manifests. |
| 3 | 2-a | | `prompts/2-…` run a | | | | |
| 4 | 2-b | | `prompts/2-…` run b | | | | |
| 5 | 3 | | `prompts/3-…` | | | | |
| 6 | | | | | | | |
| 7 | | | | | | | |

# Claims Processing

Claims processing is the core CACFP workflow. Providers record meals, sponsors validate and process claims, and reimbursement data is generated for state agencies.

## Repos Involved

| Repo | Role |
|------|------|
| **KK** | Provider meal recording, claim submission |
| **HX** | Sponsor-side claim processing (legacy VB6) |
| **DistributedProcessing** | Distributed claims processing via NServiceBus + Azure Storage |
| **ClaimsProcessor.Converted** | .NET 8 migration of claims processing logic from VB6 |
| **Centers-CX** | Center-side claims engine ("Madagascar") |
| **hx_cloudconnectionAPI** | Cloud bridge between KK and HX |

## How It Works (high level)

1. Provider records meals in KK
2. KK submits claim data to HX via hx_cloudconnectionAPI
3. HX processes claims (validation, error checking, food chart generation)
4. DistributedProcessing distributes work across NServiceBus worker nodes
5. ClaimsProcessor.Converted runs the claim logic (migrated from VB6 to .NET 8)
6. Results flow back to KK for provider visibility

*Detailed documentation for each step lives in the respective repo's `docs/` folder.*

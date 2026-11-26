# Detecting Azure-Native C2 Techniques Through Behavioural Analysis and KQL-Based Threat Hunting

## The problem

Modern APT groups, including APT28 (Fancy Bear) and POLONIUM, have moved away from attacker-owned infrastructure for command-and-control. Instead, they abuse legitimate, trusted Microsoft services (the Graph API, Azure Blob Storage) to issue commands and exfiltrate data. Because the traffic is authenticated, encrypted, and directed at genuine Microsoft infrastructure rather than a suspicious external domain, the usual network-based red flags (odd IPs, bad TLS certs, domain reputation) simply don't apply. The attacker's traffic looks exactly like normal enterprise cloud activity, because technically, it is normal cloud activity, just being used for the wrong purpose.

I wanted to find out whether this kind of abuse leaves any behavioural trace at all, and whether that trace could realistically be turned into a detection rule a SOC analyst could actually deploy.

## What I built

I designed and built a controlled, isolated Azure lab reproducing two real, documented C2 techniques, each based on an actual threat intelligence report rather than a hypothetical scenario:

- **Graph API Token-Based C2**, modelled on *Operation Graphite*, attributed to APT28 with moderate confidence by Trellix. The technique abuses OAuth-authenticated Graph API access to poll commands and return results through a Microsoft 365 mailbox.
- **Azure Blob Storage Dead-Drop C2**, modelled on the *POLONIUM* campaign (and the more recent *Operation Dragon Weave*), where the attacker uses Blob Storage containers as a dead-drop: periodically uploading commands and retrieving execution results through completely legitimate storage API calls.

The lab itself ran across two linked Microsoft cloud environments (a Microsoft 365 Developer tenant for the identity/Graph API side, and an Azure Student Subscription tenant hosting the victim VM, Blob Storage, and the Sentinel/Log Analytics workspace), with a Windows Server 2020 VM as the simulated victim endpoint. I wrote custom Python scripts to simulate three separate roles for each technique, the attacker, the victim/implant, and a benign user generating realistic background enterprise activity, so that the resulting telemetry would genuinely blend malicious and legitimate behaviour together, the same way it would in a real environment.

## Methodology

The research followed three stages:

1. **Threat intelligence profiling**: selecting only well-documented, corroborated threat reports (Microsoft Security, Trellix, Symantec, IBM X-Force) and mapping the adversary behaviour to MITRE ATT&CK techniques (T1102, T1102.001, T1528, T1071.001), rather than relying on a single vendor's technique labels.
2. **Controlled simulation and machine learning analysis**: running both C2 techniques in the lab, collecting the resulting telemetry, and using machine learning to identify which behavioural features actually separated malicious activity from the benign noise around it.
3. **Detection engineering**: converting the highest-ranked behavioural features into working KQL queries deployed and tested against the collected telemetry in Microsoft Sentinel.

## Key finding: Sentinel's built-in analytics missed both techniques entirely

Before building any custom detection, I tested how Microsoft's own native security controls handled the simulated traffic. Full telemetry was collected successfully in both cases: Entra ID sign-in logs, Azure Activity logs, Blob Storage diagnostic logs, Defender for Endpoint and Sysmon events were all ingested into Sentinel without issue. Despite that, **neither simulation triggered a single built-in security alert.** The traffic was authenticated, encrypted, and directed at genuine Microsoft infrastructure, so it simply didn't match any existing signature or anomaly rule.

That result reframes the actual problem: it isn't a visibility gap, it's a correlation gap. Sentinel could see everything happening. It just couldn't connect the dots across identity, API access, and storage operations to recognise the pattern as abuse rather than normal use.

## Machine learning results

**Graph API C2, Isolation Forest, trained on Entra ID authentication telemetry**

| Metric | Result |
|---|---|
| Accuracy | 96.81% |
| Precision | 53.85% |
| Recall | 77.78% |
| F1-score | 63.64% |
| Specificity | 97.52% |

**Azure Blob Storage C2, Random Forest, trained on StorageBlobLogs telemetry**

| Metric | Result |
|---|---|
| Accuracy | 84.48% |
| Precision | 70.33% |
| Recall | 100.00% |
| F1-score | 82.58% |
| ROC-AUC | 0.9991 |

The Blob Storage model caught every single malicious event in the test set (100% recall), at the cost of a meaningful false-positive rate that would need contextual tuning before any real deployment. The feature analysis was genuinely useful here too: the single strongest signal wasn't any one suspicious operation, but the *number of storage requests from the same client within a five-minute window*, in other words, the frequency and repetition of the polling behaviour itself was the tell, not any individual action.

For the Graph API model, the highest-ranked feature on paper ("mobile/desktop client type") turned out to be a poor real-world indicator once I thought it through, a large organisation has thousands of staff authenticating from desktop clients every day, so using that alone would generate constant false positives. Recognising that and excluding it in favour of more genuinely diagnostic features (authentication timing, device code flows, Graph access patterns) was as important a part of the analysis as the model's raw output.


## From features to detection rules

The behavioural features that actually held up were converted into working KQL logic in Sentinel. Each technique has two rules: a detection query that flags the suspicious window, and a follow-up investigation query that drills into exactly what happened inside it.

**Blob Storage dead-drop C2**
- [`blob-storage-c2-detection.kql`](./detection-rules/blob-storage-c2-detection.kql) groups `StorageBlobLogs` by caller IP over a five-minute window, then flags clients exceeding thresholds across total requests, PutBlob/GetBlob counts, and a calculated polling ratio. This isolated the three most suspicious five-minute windows in the dataset, including one with 48 requests from a single client.
<img width="979" height="412" alt="image" src="https://github.com/user-attachments/assets/ad833009-8113-4d4f-a633-38f7b11ff452" />





- [`blob-storage-c2-investigation.kql`](./detection-rules/blob-storage-c2-investigation.kql) takes a flagged window from the detection query and pulls every individual log entry inside it, surfacing 55 specific Blob operations for analyst review.
 <img width="1560" height="797" alt="image" src="https://github.com/user-attachments/assets/1e195463-d1bd-4541-ace9-8377490da51c" />



**Graph API token-based C2**
- [`graph-api-c2-detection.kql`](./detection-rules/graph-api-c2-detection.kql) correlates `SigninLogs` and `AADNonInteractiveUserSignInLogs`, looking for a device-code authentication followed by a `ResultType 50199` event within 30 seconds, followed by a successful non-interactive Graph sign-in within 120 seconds.

<img width="1527" height="587" alt="image" src="https://github.com/user-attachments/assets/b886898a-26c8-4981-a79a-8ddf9530e570" />



- [`graph-api-c2-investigation.kql`](./detection-rules/graph-api-c2-investigation.kql) expands a flagged sequence into a full chronological timeline, surfacing IPs, client applications, user agents, and result codes for validation.

  <img width="1635" height="652" alt="image" src="https://github.com/user-attachments/assets/0c8b5abf-b08b-4a02-a397-33ba187f2cd4" />

  
## Honest limitations

This is lab research, not a production-ready detection suite, and I think that distinction matters more than overselling the results would:

- Both datasets were small and scenario-specific (251 authentication events, 440 storage events). Real enterprise telemetry at scale could behave differently.
- The Blob Storage rule's false-positive rate (~24.5%) means it needs contextual tuning against an organisation's actual Blob Storage usage patterns before it's operationally useful, not a drop-in deployment.
- Neither rule could be tested as a live, scheduled Sentinel analytic rule, due to permission limits on the student tenant. Detection was validated through query results and evidence, not automated alerting in production.
- Both rules are explicitly behavioural indicators, not proof of compromise. A SOC analyst's judgement is still required to confirm a flagged event is genuinely malicious.

## What this demonstrates

This project followed the same workflow a real detection engineering or threat hunting team uses in production, just at lab scale and with full visibility into every decision along the way. I built the simulation from real, documented threat intelligence rather than a made-up scenario. I used machine learning the way it should be used here, as a tool to discover which behaviours actually matter, not as a finished detector on its own. And I turned those findings into KQL rules an analyst can actually read and trust, rather than a black-box model nobody can explain.

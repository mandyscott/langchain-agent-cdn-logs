# langchain-agent-cdn-logs
Given logs from your CDN, this agent identifies anomalies, attack patterns, or performance issues.

# Usage
A user pastes a chunk of CDN access logs (or uploads a file), and the agent returns a structured triage report: anomalies flagged, suspected attack patterns, performance issues, and recommended next steps.

# What differs between this agent and using an LLM to do a basic summary of logs?
This isn't just "summarise logs with an LLM," it's a system that decides what to investigate and uses the appropriate tools to investigate it.

A naive approach would dump logs into a prompt and ask for analysis. That breaks immediately on volume, hallucinates patterns, and doesn't scale past trivial inputs. The agentic approach instead does aggregation and statistics deterministically in code, then uses the LLM where it actually adds value: pattern interpretation, attack classification, and natural-language explanation.

# Specification v0.1 (TODOs)
* Parse & normalise — Accept just one log format, in this case Solarwinds Papertrail logs. Detect the format, parse into structured records. Pure code, no LLM.
* Statistical pre-analysis — Compute the "grind" things that don't need an LLM: top IPs by request count, status code distribution, geographic spread if you have it, request rate over time, top user agents, top URLs, cache hit ratio, response size distribution, requests per IP per second. This is the unglamorous work that makes the LLM portion actually useful.
* Anomaly detector node — More grind, flag statistical outliers using simple rules (z-scores on request rates, sudden 4xx/5xx spikes, unusual user-agent concentrations, suspicious URL patterns). Mostly deterministic. Output is a list of anomaly candidates with supporting numbers. Initially support 4 anomaly detectors.
* LLM triage node — Given the aggregations and flagged anomalies (not raw logs), the LLM classifies what's likely happening: credential stuffing, scraping, layer 7 DDoS, broken deployment, cache poisoning attempt, legitimate traffic spike, etc. Important: it works from the summaries, not raw log lines, which keeps token usage sane. Use one LLM provider.
* Investigation loop — This is the "agent" part. The LLM can call tools to dig deeper: "show me all requests from this IP," "what URLs returned 5xx in the last hour of the window," "compare this user agent's behaviour to the median." Start with four tools initially.
* Report generator — Produces a structured markdown or JSON report with findings, severity ratings, and recommended actions ("rate limit this IP range," "investigate origin health," "check WAF rules for this path"). Use raw CLI output initially.

# Implemented v0.1 (Done)

# Roadmap (Future)
* Parse & normalise — accept other common CDN log formats (Fastly's default, Cloudflare, generic Combined Log Format). Detect the format, parse into structured records. Pure code, no LLM.
* Investigation loop — A few well-chosen tools beat many shallow ones.

# Archived (Will/Can Not Build)
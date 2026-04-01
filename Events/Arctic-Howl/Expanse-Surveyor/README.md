# Expanse Surveyor

![Expanse Surveyor](images/banner.png)

## Event: Arctic Howl CTF - OffSec

## Challenge Description

Returning from a foreign realm, an Expanse Surveyor crossed back into the Cascade Expanse carrying new field notes, environmental observations, and images of systems never before cataloged. Initial debriefs showed nothing out of the ordinary. The data appeared clean.

At home, the Surveyor installed the Research Gallery application on his Android to organize and review the findings. The app was used to offload photos and annotations captured during the expedition - seemingly harmless fragments of discovery.

Within 48 hours of reconnecting to their home network, anomalies surfaced. Outbound connections appeared where none should exist. Obfuscated traffic pulsed at irregular intervals, synchronized with no known service. The home network security system flagged the activity and escalated the alert. The device was immediately quarantined.

A forensic sweep followed. Application artifacts, and network logs were preserved before the signal went dark. Whatever crossed back from the foreign expanse had already begun to adapt - learning the environment it had entered.

**Your task is to analyze the recovered artifacts and determine what was introduced, how it activated, and what it attempted to communicate inside the Cascade Expanse.**

## Challenge Files

| File | Description |
|------|-------------|
| `gallery-17-gplay-release.apk` | Suspicious Android application (42 MB) |
| `user_traffic.har` | Network traffic capture (301 MB) |

## Tools Recommended

- jadx / apktool (APK decompilation)
- HAR viewer / mitmproxy (traffic analysis)
- Protocol Buffer decoder
- Hex editor

## Progress

- Questions 1-7: Completed (7/7)

## Notes

- Large HAR file requires command-line tools (jq) due to file size (301 MB)
- Malware analysis report available as `MALWARE_ANALYSIS_REPORT.pdf`

---

*Challenge from OffSec The Gauntlet: Arctic Howl - Expanse Surveyor*
*Completed*

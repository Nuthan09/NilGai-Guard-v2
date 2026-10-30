# NilGai Guard v2 – field dashboard

Live dashboard for the four-corner NilGai Guard system (IoT Lab, CSE Department, IIT (BHU)).

**Live page:** https://nuthan09.github.io/NilGai-Guard-v2/  ·  **Demo (no hardware):** https://nuthan09.github.io/NilGai-Guard-v2/?demo

```
corner slaves A B C D ──Bluetooth──► master ──MQTT (broker.hivemq.com)──► this page
                                        └──► Google Sheets (history + labels, read back by this page)
```

## Deploy
1. Upload `index.html` to the root of this repository.
2. Settings → Pages → *Deploy from a branch* → `main` / `/ (root)` → Save.

## First use
Press **Settings** on the page:
- **Farm key** = `FARM_KEY` in the master sketch (default `iitbhu-nuthan09`)
- **Google Sheets web-app URL** + **secret** = the Apps Script deployment (see the main README §7)

Settings are stored only in your browser.

## Data
Every crossing is kept, including wrong predictions. Label them in the log (Person / NilGai / Other animal / Nothing); labels go to the master (the node learns) and to Google Sheets. **Training data → Export CSV** gives one row per crossing with all features, raw beam intervals and your label.

# Setup

Exploratory testing determined that open-source OCR libraries would not achieve sufficient accuracy to be used for CAPTCHA solving at scale. Popular open-source speech recognition libraries seemed to mostly be wrappers for closed-source models, so they were not relevant. Testing was performed on OpenAI models since Free Law already uses them extensively. The models tested were: `gpt-4.1`, `gpt-4.1-mini`, `gpt-4.1-nano`, `gpt-4o-mini`, `gpt-5-mini`, and `gpt-5-nano` for image CAPTCHAs and `gpt-4o-transcribe`, `gpt-4o-mini-transcribe`, and `whisper-1` for audio CAPTCHAs. Labeled data was created by downloading 100 CAPTCHA images and 100 CAPTCHA audio files from the SCOTUS subscription page and solving each of them by hand in `label-studio`.

Audio CAPTCHAs consist of five characters read out using the NATO phonetic alphabet, so a post-processing step was applied to the output of the transcription models, shown below.

```python
def remap_audio_output(messy: list[str]) -> list[str]:
    keys = [re.sub(r'[^\w]', '', s.lower()) for s in messy]
    out = [phonetic_map[key] if key in phonetic_map else key[0] if len(key) > 0 else '' for key in keys]
    while len(out) < 5:
        out.append('')
    return out
```

Price per CAPTCHA solve was calculated by recording the number of input and output tokens returned from each model's API call and multiplying them by the price per token from the model description page. Input to transcription models was priced as if all tokens were audio tokens since those are more expensive than text tokens and OpenAI's API responses do not differentiate between text and audio tokens. The Whisper model's pricing was calculated using the price per minute from OpenAI's website and the total time returned in the API response. Pricing is detailed in the following table.

| Model                  | Input           | Output           | Audio input     |
|------------------------|-----------------|------------------|-----------------|
| gpt-4.1                | \$2.00/M tokens | \$8.00/M tokens  | n/a             |
| gpt-4.1-mini           | \$0.40/M tokens | \$1.60/M tokens  | n/a             |
| gpt-4.1-nano           | \$0.10/M tokens | \$0.40/M tokens  | n/a             |
| gpt-4o-mini            | \$0.15/M tokens | \$0.60/M tokens  | n/a             |
| gpt-4o-mini-transcribe | \$1.25/M tokens | \$5.00/M tokens  | \$3.00/M tokens |
| gpt-4o-transcribe      | \$2.50/M tokens | \$10.00/M tokens | \$6.00/M tokens |
| gpt-5-mini             | \$0.25/M tokens | \$2.00/M tokens  | n/a             |
| gpt-5-nano             | \$0.05/M tokens | \$0.40/M tokens  | n/a             |
| whisper-1              | n/a             | n/a              | \$0.006/min     |

# Analysis

Models were initially tested against 20 CAPTCHA examples. All models achieved perfect accuracy except for `gpt-4.1-nano` which incorrectly labeled one example. The `gpt-5-mini` and `gpt-5-nano` models were determined to be nonviable due to taking an excessive number of tokens and amount of time to solve CAPTCHAs compared to other models. Max completion tokens were set to 20 for all models, but had to be disabled for the GPT-5 models for them to successfully run. The GPT-5 results are summarized in the table below for the sake of completeness.

| Model        | Time/CAPTCHA | Price/1000 CAPTCHA |
|--------------|--------------|--------------------|
| `gpt-5-mini` | 6.23s        | \$0.681            |
| `gpt-5-nano` | 8.75s        | \$0.394            |

A second analysis was run against 100 CAPTCHA examples, excluding the GPT-5 models. The results are summarized in the table below. Expected attempts is the expected number of attempts it will take the model to solve a CAPTCHA, based on its accuracy (calculated by summing a geometric series).

| Model                    | Modality | Time/CAPTCHA | Accuracy | Price/1000 CAPTCHA | Expected Attempts | Expected time/solve | Expected price/1000 solves |
|--------------------------|----------|--------------|----------|--------------------|-------------------|---------------------|----------------------------|
| `gpt-4.1`                | Image    | 1.25s        | 100%     | \$0.8200           | 1.00              | 1.25s               | \$0.8200                   |
| `gpt-4.1-mini`           | Image    | 0.72s        | 99%      | \$0.0812           | 1.01              | 0.73s               | \$0.0882                   |
| `gpt-4.1-nano`           | Image    | 0.62s        | 94%      | \$0.0357           | 1.06              | 0.66s               | \$0.0378                   |
| `gpt-4o-mini`            | Image    | 1.42s        | 100%     | \$1.3405           | 1.00              | 1.42s               | \$1.3405                   |
| `gpt-4o-transcribe`      | Audio    | 1.14s        | 100%     | \$0.7162           | 1.00              | 1.14s               | \$0.7162                   |
| `gpt-4o-mini-transcribe` | Audio    | 0.89s        | 100%     | \$0.3710           | 1.00              | 0.89s               | \$0.3710                   |
| `whisper-1`              | Audio    | 1.46s        | 100%     | \$1.0990           | 1.00              | 1.46s               | \$1.0990                   |

Based on these results, `gpt-4.1-nano` is the best model if price is the primary consideration and `gpt-4o-mini-transcribe` is the fastest and cheapest model with 100% accuracy. The `gpt-4.1-mini` model provides a good middle ground between accuracy and performance. In my opinion, performance is the primary consideration since failing a CAPTCHA may lead to the server flagging our scraper, which will cause future problems so I recommend `gpt-4o-mini-transcribe`, but I am open to other opinions.

# Commercial Alternatives

For the sake of completeness, commercial CAPTCHA solving services were also evaluated. These would have the advantage of being more resilient to potential future changes to what variety of CAPTCHA is in use, and any code would be applicable more broadly than just the SCOTUS notification subscription page. Three main services were considered: [Capsolver](https://www.capsolver.com) and [AZcaptcha](https://azcaptcha.com). Other services are available, but these were the only two found with competitive response times and pricing to OpenAI-based solving.

| Service   | \$/1,000 images | Time/solve | Accuracy | Expected attempts | Expected \$/1,000 solves | Expected time/solve |
|-----------|-----------------|------------|----------|-------------------|--------------------------|---------------------|
| AZCaptcha | $0.40           | 0.1-0.5s   | 95%      | 1.05              | $0.42                    | 0.1-0.5s            |
| Capsolver | $0.40           | <1s        | 90%      | 1.11              | $0.44                    | <1.1s               |

AZCaptcha does not offer significantly better pricing or speed than OpenAI--and anecdotally, its website looks sketchy--so I recommend against them.

Capsolver claims to be used by a variety of universities and other organizations that regularly need to perform web scraping. They also seem to have good API documentation. While they may be slightly slower than `gpt-4o-mini-transcribe` in the worst case, they have competitive pricing and are not subject to unexpectedly breaking if SCOTUS stops using Kendo CAPTCHAs.
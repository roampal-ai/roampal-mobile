# Roampal Mobile

[![License](https://img.shields.io/badge/License-Apache%202.0-blue.svg)](LICENSE)

**On-device AI with persistent memory. Powered by Gemma.**

Roampal Mobile brings the full Roampal memory sidecar pattern to your phone. Experience large language models that remember your conversations, learn from feedback, and get smarter over time -- all running locally, fully offline, with zero data sent to the cloud.

Built on the [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery) foundation (Apache 2.0), Roampal Mobile adds a persistent memory layer that follows you across every chat, image analysis, and voice transcription.


| **Install from Google Play** | **Install from App Store** |
| :--- | :--- |
| *Coming soon* | *Coming soon* |


## Core Features

* **Persistent Memory**: Every conversation is stored, embedded, and retrieved. The model remembers facts, preferences, and context across sessions.
* **AI Chat with Thinking Mode**: Multi-turn conversations with the Gemma 4 family, with optional step-by-step reasoning visibility.
* **Ask Image**: Upload images and ask questions -- vision-language understanding on-device.
* **Audio Scribe**: Transcribe and translate voice recordings in real-time.
* **Prompt Lab**: Test single-turn prompts with full parameter control.
* **Model Management & Benchmark**: Download and run models from the Gemma family, Qwen, and more. Benchmark performance on your specific hardware.
* **100% On-Device Privacy**: All inference, embeddings, and memory storage happen locally. No internet required after model download.

## Technology Highlights

*   **Google AI Edge / LiteRT LM**: Core runtime for on-device LLM inference.
*   **Roampal Memory Layer**: SQLite-based vector store with model2vec embeddings and cross-encoder reranking.
*   **Wilson Score Tracking**: Statistical confidence scoring displayed as memory metadata for transparency.
*   **TagCascade Retrieval**: Two-lane semantic search -- summaries and facts -- merged and reranked.
*   **Hugging Face Integration**: Model discovery and download.

## Development

Check out the [development notes](DEVELOPMENT.md) for instructions about how to build the app locally.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full technical architecture of the memory system.

## License

Licensed under the Apache License, Version 2.0. See the [LICENSE](LICENSE) file for details.

## Attribution

Built on [Google AI Edge Gallery](https://github.com/google-ai-edge/gallery), licensed under Apache 2.0.  
Powered by [Gemma](https://ai.google.dev/gemma) models.

## Useful Links

*   [Roampal Core](https://github.com/roampal/roampal-core)
*   [Google AI Edge Documentation](https://ai.google.dev/edge)
*   [Hugging Face LiteRT Community](https://huggingface.co/litert-community)

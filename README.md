# CognitiveCart

**A hands-free grocery shopping assistant for Meta smart glasses: plan a shopping list by chat, then get spoken, profile-aware product advice in the store from shelf photos and barcode scans.**

![React Native](https://img.shields.io/badge/React_Native-0.84-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![Swift](https://img.shields.io/badge/Swift-iOS_bridge-F05138?logo=swift&logoColor=white)
![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-009688?logo=fastapi&logoColor=white)
![SQLite](https://img.shields.io/badge/SQLite-003B57?logo=sqlite&logoColor=white)

> **About this fork.** This is Dev Choudary's fork of [Akash2971/CognitiveCart](https://github.com/Akash2971/CognitiveCart), which is the upstream project and where development happens. At the time of writing the fork's `main` branch matches upstream `main` exactly; apart from this README, the fork contains no changes of its own.

---

## Contents

- [Overview](#overview)
- [How a trip works](#how-a-trip-works)
- [Features](#features)
- [Architecture](#architecture)
- [Tech stack](#tech-stack)
- [UI mockups](#ui-mockups)
- [Getting started](#getting-started)
- [Configuration](#configuration)
- [Backend API](#backend-api)
- [Product catalog](#product-catalog)
- [Repository layout](#repository-layout)
- [Status and known gaps](#status-and-known-gaps)
- [Credits](#credits)
- [License](#license)

## Overview

Standing in front of a grocery shelf can mean too many choices, labels that are hard to compare, and products that are hard to find. CognitiveCart pairs a phone app with Ray-Ban Meta glasses so a shopper can ask questions out loud, have the glasses' camera look at the shelf, and hear a short answer back.

The project is built around a "don't let the model make up facts" approach: nutrition numbers come from a local product catalog or from [Open Food Facts](https://world.openfoodfacts.org/), and the language model is asked to reason over that data rather than recall it. Answers are kept to a couple of spoken sentences.

Recommendations are personalised using a profile of goals (e.g. weight loss, muscle gain, heart health), dietary restrictions (e.g. gluten-free, dairy-free, nut-free) and priorities (e.g. low sugar, high protein, low fat).

## How a trip works

1. **Set a profile** on the Profile tab.
2. **Plan at home** on the Chat tab: describe a meal, answer a clarifying question or two, and pick which suggested ingredients go on your list.
3. **Start a trip** on the Trip tab. From there you have four controls:
   - **Hold to talk:** ask a grocery question by voice. The assistant answers out loud and, when it fits, tells you which of the other buttons to use.
   - **Location:** shows which aisle a product category is in.
   - **Start Assistance:** the glasses take a short burst of photos of the shelf in front of you; the backend identifies the category and products, then recommends the top three picks from the catalog for your profile.
   - **Barcode:** scan one or more products with the phone camera; the assistant gives a take / skip / consider verdict for a single item, or picks a winner when you've scanned several.
4. **End the trip** to see a summary: how long it took, how many barcode scans and shelf-assistance requests you made, and which kinds of "cognitive load" came up (search for Location, choice overload for Start Assistance, comprehension or comparison for barcode checks).

## Features

- **Recipe-to-list planning chat.** The assistant asks about the dish variation and dietary needs before proposing ingredients, offers dish ideas when you're undecided, and adds items you name directly.
- **Shopping list** with check-off, plus a button to upload a photo of a store map, which the backend parses into aisles and landmarks.
- **Voice in, voice out.** Speech-to-text via `@react-native-voice/voice` and spoken replies via `react-native-tts`.
- **Glasses camera capture** through a native iOS module built on Meta's Wearables Device Access Toolkit (`MWDATCore`, `MWDATCamera`), exposed to JavaScript as `WearablesModule`.
- **Two-step shelf analysis.** First a vision call lists the category and readable products; detected names are fuzzy-matched against the catalog (RapidFuzz). Then a separate text-only call ranks the catalog products for that category against your profile, with a score and one-line reason per pick. Unsupported categories get a short general recommendation instead.
- **Barcode lookup.** On the phone, `react-native-vision-camera` reads EAN-13/8, UPC-A/E, Code 39 and Code 128 codes. The backend also has an image-based decoder (zxing-cpp with several contrast, sharpening and crop/upscale passes) for frames from the glasses. Product data is fetched from Open Food Facts and cached for the session, and you can attach a shelf price to a scanned product.
- **Store locations** for each supported category, read from the database without any AI call.

## Architecture

```
 Ray-Ban Meta glasses ──(Meta DAT SDK, iOS)──┐
                                             ▼
                              ShopifyApp (React Native)
                    Chat · List · Trip · Profile tabs, Zustand store
                                             │  HTTP (JSON, base64 images)
                                             ▼
                               backend/ (FastAPI, Python)
                ┌────────────────────┬───────────────────────┬─────────────────┐
                ▼                    ▼                       ▼                 ▼
   OpenAI-compatible VLM     SQLite (products.db)     Open Food Facts     zxing-cpp
   (e.g. served by vLLM)     catalog, aisles,         product API         barcode decoding
                             profile, scans
```

The backend talks to the model through the `openai` Python client pointed at any OpenAI-compatible endpoint (`VLLM_BASE_URL`). The design notes in `docs/newplan/agent_architecture.md` target Qwen 3 8B. Each button in the app maps to its own backend agent rather than one agent routing between tools.

## Tech stack

| Layer | Tools |
| --- | --- |
| Mobile app | React Native 0.84, React 19, TypeScript, React Navigation (bottom tabs), Zustand |
| Device features | react-native-vision-camera, @react-native-voice/voice, react-native-tts, react-native-image-picker |
| Glasses | Meta Wearables Device Access Toolkit for iOS (`meta-wearables-dat-ios` 0.3.0 via Swift Package Manager), Swift/Objective-C bridge |
| Backend | Python 3.11, FastAPI, Uvicorn, Pydantic |
| AI | `openai` client against an OpenAI-compatible vision-language model server |
| Data | SQLite, Open Food Facts API, RapidFuzz, Pillow, zxing-cpp |

## UI mockups

These design mockups live in `docs/uiscreens/`. They show an earlier three-tab layout (Chat, List, Capture); the current app has Chat, List, Trip and Profile tabs.

| Chat | List | Capture (in-store) |
| --- | --- | --- |
| ![Chat tab mockup](docs/uiscreens/chat_tab.png) | ![List tab mockup](docs/uiscreens/list_tab.png) | ![Capture tab mockup](docs/uiscreens/capture_tab_active.png) |

## Getting started

### Prerequisites

- **Backend:** Python 3.11 (pinned in `backend/.python-version`) and access to an OpenAI-compatible model server with vision support.
- **App:** Node.js 22.11 or newer, the React Native CLI toolchain, and for iOS: Xcode, Ruby/Bundler and CocoaPods.
- **Glasses (optional for planning features):** Ray-Ban Meta glasses paired through the Meta AI app with developer mode enabled, as required by Meta's Wearables Device Access Toolkit.

### 1. Run the backend

```sh
cd backend
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

# create backend/.env (see Configuration), then:
uvicorn main:app --host 0.0.0.0 --port 8080
```

Run it from inside `backend/`, because the default database path (`products.db`) is relative. Check it's up with `GET http://<host>:8080/health`.

The repo also contains `pyproject.toml` and `uv.lock` for [uv](https://docs.astral.sh/uv/), but that dependency list is missing Pillow, zxing-cpp and NumPy, which the barcode code needs, so `requirements.txt` is the more complete option.

A `Procfile` (`web: uvicorn main:app --host 0.0.0.0 --port $PORT`) is included for Procfile-based hosts.

### 2. Point the app at the backend

Edit `ShopifyApp/src/config.ts` and set `BACKEND_URL` to an address your phone can reach, such as your computer's LAN IP:

```ts
export const BACKEND_URL = 'http://<your-computer-ip>:8080';
```

### 3. Run the mobile app

```sh
cd ShopifyApp
npm install

# iOS (first time, and whenever native dependencies change)
bundle install
cd ios && bundle exec pod install && cd ..

npm start          # Metro bundler
npm run ios        # in a second terminal
```

`npm run android` is also available, but the glasses bridge is only implemented for iOS (see [Status and known gaps](#status-and-known-gaps)).

Other scripts: `npm run lint` (ESLint) and `npm test` (Jest; there is a single render smoke test).

## Configuration

Backend settings are read from the environment or from `backend/.env` (gitignored):

| Variable | Default | Purpose |
| --- | --- | --- |
| `VLLM_BASE_URL` | empty | Base URL of the OpenAI-compatible model server. Can also be changed while running via `POST /config/vlm_url`. |
| `VLLM_MODEL` | none | Model name sent with every request. |
| `VLLM_API_KEY` | `none` | API key for the model server, if it needs one. |
| `DB_PATH` | `products.db` | SQLite database file. |

App setting: `BACKEND_URL` in `ShopifyApp/src/config.ts`. For the glasses SDK, `ios/ShopifyApp/Info.plist` contains an `MWDAT` block with the app-link scheme `shopifyapp://` and an empty `MetaAppID`.

## Backend API

Routes mounted by `backend/main.py`:

| Method and path | Used for |
| --- | --- |
| `GET /health` | Status, active model name and model server URL |
| `GET` / `POST /config/vlm_url` | Read or change the model server URL at runtime |
| `POST /chat` | Planning chat: clarifying questions, dish ideas, steps, tips, ingredient proposals |
| `POST /ptt` | Push-to-talk question during a trip (text only, no image) |
| `GET /locations`, `GET /location/{category}` | Aisle and landmark for supported categories |
| `GET` / `POST /user_profile` | Read or save goals, restrictions and priorities |
| `POST /shelf_scan` | Shelf photos → detected products and top-3 catalog recommendations |
| `POST /scan_barcode` | Decode a barcode from an image and look the product up |
| `GET /lookup_barcode/{barcode}` | Look up a barcode string |
| `GET` / `DELETE /scanned_products` | List or clear this session's scanned products (the app clears them on launch) |
| `PATCH /scanned_products/{barcode}/price` | Record a shelf price for a scanned product |
| `POST /barcode_analyze` | Verdict for one product, or best pick among several |
| `POST /product_info` | Spoken nutrition summary and comparison for given products |
| `POST /store_map` | Parse an uploaded store-map photo into aisles, sections and landmarks |
| `POST /test/barcode` | Debug endpoint for barcode decoding and lookup |
| `DELETE /agent_state` | No-op reset endpoint |

FastAPI's interactive docs are available at `/docs` while the server runs.

## Product catalog

`backend/products.db` is committed and already contains:

- a **catalog** of 84 products across seven categories: yogurt, milk, cheese, jam, cereal, granola and pasta sauce;
- **store locations** (aisle and landmark notes) for those seven categories;
- a default **user profile**.

The catalog was built by `backend/seed_catalog.py`, which decodes the barcode photos in `backend/barcode images/<category>/`, fetches each product from Open Food Facts, and inserts it. Progress is checkpointed in `seed_progress.json` so a re-run only retries failures. Barcodes that Open Food Facts didn't know are listed in `backend/not_in_off.txt`.

To rebuild it yourself, run `python seed_catalog.py` from `backend/`.

## Repository layout

```
CognitiveCart/
├── ShopifyApp/                  # React Native app
│   ├── App.tsx                  # tab navigator; loads profile and clears session scans on launch
│   ├── WearablesModule.ts       # JS wrapper for the native glasses module
│   ├── src/
│   │   ├── config.ts            # BACKEND_URL
│   │   ├── store/shoppingStore.ts
│   │   └── screens/             # Chat, ShoppingList, Trip, Profile, BarcodeScan, LiveCapture
│   ├── ios/                     # Xcode project, Podfile, Swift/ObjC WearablesModule and stream preview
│   └── android/                 # standard React Native Android project
├── backend/                     # FastAPI service
│   ├── main.py                  # app setup and router registration
│   ├── config.py                # env vars and model client
│   ├── database.py · schema.sql # SQLite access
│   ├── models.py                # request/response models
│   ├── agent_state.py           # in-memory session state
│   ├── routers/                 # one module per feature
│   ├── seed_catalog.py · seed.py
│   ├── products.db
│   └── barcode images/          # source photos for the catalog
├── docs/                        # plans, agent design notes, API contract, UI mockups
├── buildorder                   # working task list
└── test_barcode.py              # CLI helper that posts an image to /test/barcode
```

## Status and known gaps

This is a working prototype. Things worth knowing:

- **Glasses support is iOS-only.** The native `WearablesModule` exists only in the iOS project, so the glasses camera features won't work on Android.
- **Legacy code is still present.** `LiveCaptureScreen.tsx` isn't reachable from the app's navigation, and the backend routers it calls (`active.py`, `passive.py`) are not registered in `main.py`, nor are `detect.py`, `dwell.py` and `vision_chat.py`. `seed.py` and `backup.py` also belong to earlier iterations.
- **Store-map uploads aren't used by the current trip flow.** The parsed map is kept in memory, but only the unregistered `active.py` reads it; the Location button uses the `store_locations` table instead.
- **Two dependency lists disagree** (`requirements.txt` vs `pyproject.toml`), as noted above.
- **Backend URL is hard-coded** in `src/config.ts` rather than configurable at build time.
- **State is local and single-user.** The profile, scans and map live in one SQLite file and in process memory.
- The design notes in `docs/` describe some ideas (such as thumbs-up/down feedback and active/passive modes) that belong to the earlier Capture-tab design.

## Credits

CognitiveCart was created by [Akash Elumalai (Akash2971)](https://github.com/Akash2971); see the [upstream repository](https://github.com/Akash2971/CognitiveCart). Product data comes from [Open Food Facts](https://world.openfoodfacts.org/). Glasses integration uses Meta's [Wearables Device Access Toolkit for iOS](https://github.com/facebook/meta-wearables-dat-ios).

## License

No license has been specified for this project, either here or upstream. Until one is added, all rights are reserved by the original author.

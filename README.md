# Recomendador Musical — AI Music Recommendation Assistant

A conversational music recommender developed as an undergraduate thesis project (*trabajo de título*). Users chat with an AI "music expert" that reads their listening profile from streaming-platform APIs and uses a Large Language Model (LLM) to suggest songs, albums, artists and genres. Every recommendation comes with a story: band history, the meaning of the lyrics, the cultural movement behind it, and why it fits the user.

> The user interface and the assistant's answers are in Spanish.

---

## Table of Contents

- [Purpose](#purpose)
- [Scope](#scope)
- [Architecture](#architecture)
- [How It Works](#how-it-works)
- [Repository Structure](#repository-structure)
- [Dependencies](#dependencies)
- [Setup](#setup)
- [API Reference](#api-reference)
- [Data Model (Firestore)](#data-model-firestore)
- [Known Limitations and Work in Progress](#known-limitations-and-work-in-progress)
- [License](#license)

---

## Purpose

Most streaming recommenders show a list of tracks with no explanation. This project tests a different approach: a **narrative, emotionally aware recommender** that:

- **Encourages discovery.** It suggests music that is *compatible with but new to* the user, instead of repeating artists they already listen to.
- **Explains every recommendation**, so the user understands why something was suggested.
- **Responds to mood.** It acknowledges emotions the user expresses and suggests music that suits or shifts that mood.
- **Collects user feedback** (ratings and short qualitative answers) to evaluate the experience as part of the thesis research.

## Scope

**Included**

- A chat interface for asking about music, with persistent, titled conversations.
- Spotify login (OAuth 2.0). The user's top artists, genres and recently played tracks become context for the LLM.
- A Spotify link card (URL and cover art) for each concrete recommendation (track, album, artist, or a playlist for a genre).
- Conversation management: list, reopen, delete one, or delete all.
- Logout, which removes Spotify credentials and keeps chat history.
- A feedback form (1–5 rating, "pleasant interaction?", "motivated you to explore?", free-text comments).
- Guardrails: off-topic requests are redirected back to music, and the model is told not to invent information.

**Not included / out of scope (current version)**

- Audio playback. Recommendations link out to Spotify.
- Production deployment, hardening and authentication beyond the Spotify user ID.
- Last.fm and YouTube Music login. These have UI and client code in the frontend but no backend endpoints yet (see [Known Limitations](#known-limitations-and-work-in-progress)).

## Architecture

```
┌────────────────────┐      HTTP/JSON      ┌──────────────────────────┐
│  Frontend          │  ─────────────────▶ │  Backend (FastAPI)       │
│  React 19 + Vite   │ ◀─────────────────  │                          │
│  localhost:5173    │                     │  main.py        (routes) │
└────────────────────┘                     │  chat_logic.py  (LLM)    │
          ▲                                │  spotify_service.py      │
          │ redirect ?uid=<spotify_id>     │  database.py (Firestore) │
          │                                └───┬─────────┬────────┬───┘
          │                                    │         │        │
   ┌──────┴──────────┐               ┌─────────▼──┐ ┌────▼─────┐ ┌▼───────────────┐
   │ Spotify Accounts│◀── OAuth ─────│ Spotify    │ │ Google   │ │ Firebase       │
   │ (login page)    │               │ Web API    │ │ Gemini   │ │ Firestore      │
   └─────────────────┘               └────────────┘ │ 2.5 Flash│ └────────────────┘
                                                    └──────────┘
```

The two parts are separate Git repositories, included here as nested repos:

| Folder      | Repository                                                   | Stack                              |
|-------------|--------------------------------------------------------------|------------------------------------|
| `backend/`  | https://github.com/MaikAF/recomendador-musical-back          | Python, FastAPI, LangChain, Gemini |
| `frontend/` | https://github.com/MaikAF/recomendador-musical-front         | React 19, Vite 7, Axios            |

## How It Works

### 1. Authentication (Spotify OAuth)

1. The frontend calls `GET /login`. The backend creates a one-time OAuth `state` value (stored in Firestore for 10 minutes, as CSRF protection) and returns a Spotify authorization URL with the scopes `user-read-private user-read-email user-top-read user-read-recently-played`.
2. The user approves access on Spotify, and Spotify redirects to `GET /callback?code=...&state=...`.
3. The backend checks and consumes the `state`, exchanges the code for tokens, gets the user's Spotify ID and display name, and stores them in Firestore (`users/{user_id}`). Spotify tokens never leave the server.
4. The backend creates a random **session token**, valid for 30 days. Firestore stores only its SHA-256 hash (`sessions/{hash}`). The backend then redirects to `<FRONTEND_URL>#session=<token>`. The token travels in the URL fragment, so it isn't sent to servers or written to access logs.
5. The frontend saves the token in `localStorage`, removes it from the address bar, and sends it as `Authorization: Bearer <token>` on every request. The backend gets the user from the session and never trusts a `user_id` sent by the client.
6. When a Spotify token is about to expire (less than 60 s left), `spotify_service.get_valid_sp_client` refreshes it and saves the new one.

### 2. Chat pipeline (`POST /chat`)

1. **Conversation.** If the request has no `conversation_id`, the backend creates one.
2. **Persist the user message** to the conversation's message array.
3. **Build context:**
   - *Conversation memory.* Instead of resending the full transcript, the backend keeps a list of short summaries (`summary_history`), one per exchange ("Request: … Response: …"), and passes them to the model. This keeps prompts small.
   - *Listening profile.* The 15 top artists (with genres) and the 10 most recently played tracks from the Spotify API. If the profile can't be loaded, the model is told to ask about the user's taste.
4. **LLM call.** A LangChain chain (`PromptTemplate | Gemini 2.5 Flash | StrOutputParser`, temperature 0.7) receives a Spanish system prompt that defines the assistant's personality and rules, and requires a strict JSON reply:

   ```json
   {
     "conversational_response": "Short intro ||| detailed explanation",
     "recommendation_type": "artist | album | track | genre | info",
     "recommendation_query": "Exact name to search on Spotify, or null",
     "history_summary": "Petición: ... Respuesta: ...",
     "conversation_title": "Short title (only on the first message), or null"
   }
   ```

   The backend strips any Markdown code fences from the reply and parses the JSON. If parsing fails, it returns a safe fallback message.
5. **Spotify enrichment.** If the reply recommends something concrete, the backend searches Spotify with the Client Credentials flow and returns the first result's URL and image. For genres it searches for a playlist named `"The Sound of <genre>"`. If nothing is found, it adds a note to the reply.
6. **Persist** the summary, the title (first message only) and the bot's reply. Return `{ response, conversation_id, spotify_data }`.

### 3. Rendering (frontend)

- `useChat.js` holds all state and API calls.
- `BotMessage.jsx` splits the reply on `|||`. It shows the short intro first (with a typewriter effect), then the detailed explanation (rendered as Markdown), then a `SpotifyCard` with the link and cover art.
- `SideBar.jsx` lists past conversations. `SettingsModal`, `FeedbackModal` and `AuthModal` handle settings and history deletion, the feedback survey, and login.

## Repository Structure

```
Recomendador-musical/
├── README.md
├── LICENSE                     # Apache 2.0
├── backend/
│   ├── main.py                 # FastAPI app, routes, CORS, Spotify OAuth
│   ├── chat_logic.py           # Prompt, Gemini/LangChain chain, JSON parsing
│   ├── spotify_service.py      # Token refresh, user profile context, search
│   ├── database.py             # Firestore access (users, conversations, feedback)
│   └── .gitignore              # ignores .env, serviceAccountKey.json, venv
└── frontend/
    ├── index.html
    ├── package.json
    ├── vite.config.js
    └── src/
        ├── App.jsx
        ├── hooks/useChat.js            # State + API client
        └── components/
            ├── ChatArea.jsx
            ├── SideBar.jsx
            ├── modals/  (AuthModal, FeedbackModal, SettingsModal)
            └── ui/      (BotMessage, SpotifyCard, PreviewCard, TypeWriter, MessageSkeleton)
```

## Dependencies

### External services

| Service                | Used for                                      | What you need                                  |
|------------------------|-----------------------------------------------|------------------------------------------------|
| Spotify Web API        | OAuth login, listening profile, search        | An app in the Spotify Developer Dashboard      |
| Google Gemini API      | LLM (`gemini-2.5-flash`)                      | An API key from Google AI Studio               |
| Firebase Firestore     | Users, tokens, conversations, feedback        | A Firebase project and a service-account key   |

### Backend (Python 3.12)

The backend has no `requirements.txt` yet. These are the packages it imports:

| Package                  | Purpose                                 |
|--------------------------|-----------------------------------------|
| `fastapi`, `uvicorn`     | Web framework and ASGI server           |
| `pydantic`               | Request models (ships with FastAPI)     |
| `spotipy`                | Spotify Web API client and OAuth        |
| `langchain-core`         | Prompt templates and chains             |
| `langchain-google-genai` | Gemini chat model integration           |
| `firebase-admin`         | Firestore access                        |
| `python-dotenv`          | Loads `.env` variables                  |

### Frontend (Node.js 20+ recommended)

- **Runtime:** `react` 19, `react-dom`, `react-router-dom` 7, `axios`, `react-markdown`, `lucide-react`, `@icons-pack/react-simple-icons`
- **Tooling:** `vite` 7, `@vitejs/plugin-react`, `eslint` 9 (with the React Hooks and React Refresh plugins)

## Setup

### 1. Clone

`backend/` and `frontend/` are separate repositories. If they come up empty after cloning this repo, clone them directly:

```bash
git clone https://github.com/MaikAF/recomendador-musical-back.git backend
git clone https://github.com/MaikAF/recomendador-musical-front.git frontend
```

### 2. Configure external services

1. **Spotify.** Create an app at <https://developer.spotify.com/dashboard> and add the redirect URI `http://127.0.0.1:8000/callback`.
2. **Gemini.** Create an API key at <https://aistudio.google.com/>.
3. **Firebase.** Create a project and enable Firestore. Then go to *Project settings → Service accounts → Generate new private key* and save the file as `backend/serviceAccountKey.json`.

### 3. Backend

Create `backend/.env`:

```env
SPOTIFY_CLIENT_ID=your_spotify_client_id
SPOTIFY_CLIENT_SECRET=your_spotify_client_secret
SPOTIFY_REDIRECT_URI=http://127.0.0.1:8000/callback
GOOGLE_API_KEY=your_gemini_api_key
FRONTEND_URL=http://127.0.0.1:5173   # optional; used for CORS and the post-login redirect
```

Install the dependencies and start the server:

```bash
cd backend
python -m venv venv
# Windows: venv\Scripts\activate   |   macOS/Linux: source venv/bin/activate
pip install fastapi uvicorn spotipy langchain-core langchain-google-genai firebase-admin python-dotenv
uvicorn main:app --reload --host 127.0.0.1 --port 8000
```

The API runs at `http://127.0.0.1:8000`, with interactive docs at `/docs`.

### 4. Frontend

```bash
cd frontend
npm install
npm run dev        # serves on http://127.0.0.1:5173
```

The API URL defaults to `http://127.0.0.1:8000`. To change it, create `frontend/.env`:

```env
VITE_API_URL=http://127.0.0.1:8000
```

> Use `127.0.0.1`, not `localhost`, for both servers (or set `FRONTEND_URL` to match). The OAuth callback redirects to `FRONTEND_URL`, and the session in `localStorage` only works on that origin.
>
> **Never commit `.env`, `serviceAccountKey.json` or spotipy's `.cache` file.** All three are listed in `backend/.gitignore`.

## API Reference

Endpoints marked 🔒 require the header `Authorization: Bearer <session_token>` and return `401` without a valid session. The user is always taken from the session. Conversation IDs that don't belong to the user return `404`.

| Method | Endpoint                         | Description                                              |
|--------|----------------------------------|----------------------------------------------------------|
| GET    | `/`                              | Health check                                             |
| GET    | `/login`                         | Returns the Spotify authorization URL (with `state`)     |
| GET    | `/callback?code=&state=`         | OAuth callback: checks `state`, stores tokens, redirects with `#session=` |
| GET    | `/me` 🔒                         | `{ user_id, display_name }` of the current session       |
| POST   | `/new_chat` 🔒                   | → `{ conversation_id }`                                  |
| POST   | `/chat` 🔒                       | `{ message, conversation_id? }` → `{ response, conversation_id, spotify_data }` |
| GET    | `/history/{conversation_id}` 🔒  | Messages of a conversation                               |
| GET    | `/conversations` 🔒              | Summaries (id, title, date) of all of the user's chats   |
| DELETE | `/conversations/{conversation_id}` 🔒 | Deletes one conversation                            |
| DELETE | `/conversations` 🔒              | Deletes all of the user's conversations                  |
| DELETE | `/logout` 🔒                     | Removes stored Spotify tokens and all of the user's sessions; keeps history |
| POST   | `/feedback` 🔒                   | `{ rating (1–5), pleasant_interaction, motivated_exploration, comments }` |

## Data Model (Firestore)

```
sessions/{sha256(token)}              # user_id, created_at, expires_at (30 days)
oauth_states/{state}                  # expires_at (10 minutes, single use)

users/{user_id}                       # display_name + Spotify token fields (access_token, refresh_token, expires_at, ...)
  └── conversations/{conversation_id}
        conversation_id: string
        fecha_inicio:    timestamp
        resumen:         string        # conversation title
        mensajes:        [{ role: "user" | "bot", text, timestamp }]
        summary_history: [string]      # compact memory passed to the LLM

feedback/{auto_id}
  User_Hash, Rating,
  Respuestas_cualitativas: { Interacción_agradable, motivacion_exploracion, comentarios_adicionales },
  Fecha_de_creación
```

## Known Limitations and Work in Progress

- **Frontend calls endpoints the backend doesn't have yet:** `GET /user/{uid}` (display name), `POST /login/lastfm`, `GET /login/ytmusic`. Last.fm and YouTube Music login won't work until these are implemented.
- **Spotify data isn't saved with messages.** `/history` returns only `role` and `text`, so Spotify cards don't reappear when an old conversation is reopened.
- **Usage limits are drafted but not enforced.** `check_conversation_limits` (10 user messages per chat) and `check_new_chat_limit` (3 chats per user) exist in `database.py` but are never called.
- **Chat requires login.** Every data endpoint needs a session, so users without a Spotify account can't chat yet. The UI already disabled chat input for users who weren't logged in.
- **Not production-hardened.** There is no rate limiting, sessions live in `localStorage` (readable by any XSS), and expired session and `state` documents are deleted only when they are read, never by a cleanup job.
- **Repository hygiene.** The sub-projects are committed as gitlinks without a `.gitmodules` file. Neither sub-project has a `requirements.txt` or automated tests yet.

## License

Distributed under the **Apache License 2.0**. See [`LICENSE`](LICENSE).

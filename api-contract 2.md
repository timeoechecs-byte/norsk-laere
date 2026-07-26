# Contrat d'API

Base : `/api/v1`. JSON uniquement. Auth par cookie de session `HttpOnly; Secure; SameSite=Lax`.
Toutes les entrées et sorties sont typées par des schémas Zod dans `packages/shared/schemas/`.

**Principe offline-first** : le client fonctionne sans serveur. L'API sert uniquement
à la synchronisation, au compte utilisateur et aux services IA distants (fallback).

---

## Format de réponse

Succès :
```json
{ "data": { ... } }
```

Erreur :
```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Message lisible en français",
    "details": { "field": "email", "issue": "invalid_format" }
  }
}
```

Codes d'erreur : `VALIDATION_ERROR` (400), `UNAUTHORIZED` (401), `FORBIDDEN` (403),
`NOT_FOUND` (404), `CONFLICT` (409), `RATE_LIMITED` (429), `INTERNAL_ERROR` (500),
`SERVICE_UNAVAILABLE` (503).

---

## Auth

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| POST | `/auth/register` | `{email, password, displayName}` | `{user, session}` |
| POST | `/auth/login` | `{email, password}` | `{user, session}` |
| POST | `/auth/logout` | — | `204` |
| GET | `/auth/me` | — | `{user, profile, settings}` |
| POST | `/auth/refresh` | — | `{session}` |
| POST | `/auth/request-reset` | `{email}` | `204` (toujours, anti-énumération) |
| POST | `/auth/reset` | `{token, password}` | `204` |

Mot de passe : ≥ 10 caractères. Rate limit login : 5 tentatives / 15 min / IP.

---

## Profil

| Méthode | Route | Notes |
|---|---|---|
| GET | `/profile` | |
| PATCH | `/profile` | champs de `user_profiles` |
| PATCH | `/settings` | champs de `user_settings` |
| DELETE | `/account` | suppression RGPD, purge sous 30 j |
| GET | `/account/export` | export JSON de toutes les données utilisateur |

---

## Contenu

Le contenu est livré avec l'app. Ces routes servent aux mises à jour incrémentales.

| Méthode | Route | Notes |
|---|---|---|
| GET | `/content/manifest` | `{version, checksums: {levels, lessons, vocabulary, ...}}` |
| GET | `/content/bundle?since=<version>` | delta de contenu, gzip |
| GET | `/content/audio/:id` | fichier audio, `Cache-Control: immutable` |

---

## Synchronisation

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| POST | `/sync/push` | `{ops: SyncOp[], clientVersion}` | `{accepted[], rejected[], serverTime}` |
| GET | `/sync/pull?since=<iso>` | — | `{changes: SyncOp[], serverTime}` |

```ts
type SyncOp = {
  id: string;              // idempotence
  table: 'user_lesson_progress' | 'user_vocabulary' | 'user_errors'
       | 'user_stats' | 'user_streaks' | 'user_passport_stamps'
       | 'user_achievements' | 'user_profiles' | 'user_settings';
  recordId: string;
  operation: 'insert' | 'update' | 'delete';
  payload: Record<string, unknown>;
  updatedAt: string;       // ISO 8601
};
```

Push idempotent par `SyncOp.id`. Max 500 ops par requête.

---

## IA (fallback distant uniquement)

Utilisé seulement si le LLM local est indisponible. Toujours dégradable.

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| POST | `/ai/chat` | `{conversationId?, message, level, scenarioTag?}` | SSE stream |
| POST | `/ai/correct` | `{text, level, context?}` | `{corrections: Correction[], rewritten}` |
| POST | `/ai/explain` | `{question, context, level}` | `{explanationFr, examples[], relatedRuleIds[]}` |
| POST | `/ai/generate-exercise` | `{type, level, focusIds[]}` | `Exercise` |

```ts
type Correction = {
  span: [number, number];       // indices dans le texte source
  original: string;
  suggestion: string;
  type: 'grammar' | 'vocabulary' | 'spelling' | 'word_order' | 'style';
  explanationFr: string;        // le « Pourquoi ? »
  frenchPitfall?: string;       // si erreur typique d'un francophone
  grammarRuleId?: string;
};
```

Rate limit IA : 60 requêtes / heure / utilisateur. En cas de dépassement,
le client bascule silencieusement sur le mode local ou désactive la fonction.

---

## Prononciation

| Méthode | Route | Corps | Réponse |
|---|---|---|---|
| POST | `/pronunciation/score` | `multipart` : audio + `{expectedText, level}` | `PronunciationScore` |

```ts
type PronunciationScore = {
  overall: number;              // 0-100
  words: Array<{
    word: string;
    score: number;
    ipaExpected: string;
    ipaActual: string;
    issue?: 'vowel_length' | 'tone' | 'consonant' | 'stress' | 'missing';
    tipFr?: string;
  }>;
  transcript: string;
};
```

---

## TTS

| Méthode | Route | Notes |
|---|---|---|
| GET | `/tts?text=...&voice=...&speed=...` | audio/mpeg, cache 30 j |

Le client tente d'abord Piper local, puis cette route, puis Web Speech API.

---

## Santé

`GET /health` → `{status, version, services: {db, llm, tts, stt}}`

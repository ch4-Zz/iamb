# Spec 0001 — Transcription des messages vocaux (speech-to-text)

Statut : **Proposition** — v2
Cible : `iamb` 0.0.12-alpha.x (fork personnel, branche `local/beeper`)
Auteur : —

---

## 1. Résumé

Ajouter à iamb la capacité de :

1. **détecter** qu'un message reçu contient un fichier joint ;
2. **identifier** que cette pièce jointe est un **message vocal** (note vocale WhatsApp
   relayée par un bridge, vocal Matrix natif, vocal Signal/Telegram…) et non un simple
   fichier audio (musique, enregistrement partagé) ;
3. **télécharger** le média (y compris s'il est chiffré E2EE) ;
4. l'**envoyer à un moteur speech-to-text** — API distante compatible OpenAI, serveur local
   (`whisper-server`), ou **inférence embarquée dans iamb** via la feature Cargo
   `whisper-local` (§6.2) ;
5. **afficher la transcription** dans le scrollback, sous le message, avec gestion d'états
   (en attente / en cours / terminé / erreur) et mise en cache.

La fonctionnalité est **désactivée par défaut** : elle envoie potentiellement le contenu de
conversations chiffrées à un tiers (cf. §9).

---

## 2. Motivation

Les notes vocales sont inconsultables depuis un client TUI : au mieux `:download` puis un
lecteur externe, ce qui casse le flux de lecture et est impossible en SSH sans audio. Or les
bridges (mautrix-whatsapp, mautrix-signal, Beeper) relaient massivement des vocaux `.ogg`.
Aujourd'hui iamb affiche seulement :

```
[Attached Audio: Message vocal.ogg (12.4 kB)]
```

Objectif : afficher en plus le texte prononcé.

---

## 3. État de l'existant (code actuel)

| Élément | Emplacement | Réutilisable |
|---|---|---|
| Action de téléchargement | `src/windows/room/chat.rs:156` (`MessageAction::Download`) | Logique de download média (`media.get_media_content`) à factoriser |
| Déclaration de l'action | `src/base.rs:101` (`MessageAction::Download`) | Modèle pour `MessageAction::Transcribe` |
| Commandes `:download` / `:open` | `src/commands.rs:1010` / `:1028` | Modèle pour `:transcribe` |
| Tâches asynchrones de fond | `src/worker.rs:715` (`WorkerTask::LoadImage`) | Modèle exact pour `WorkerTask::Transcribe` |
| Gestionnaire d'état async + rendu | `src/preview.rs` (`PreviewManager`, `ImageStatus`) | Modèle exact pour `TranscriptionManager` |
| Rendu texte des pièces jointes | `src/message/mod.rs:506` (`display_file_name!`), `:541` (`content_filename`) | Point d'injection du rendu |
| Struct message | `src/message/mod.rs:912` (`Message`, champ `downloaded`) | |
| Config (tunables/dirs) | `src/config.rs:879` (`TunableValues`), `:1118` (`DirectoryValues`) | Ajout d'une section `[transcription]` |
| Erreurs | `src/base.rs:756` (`IambError`) | Ajout de variantes |

Dépendances déjà présentes : `matrix-sdk 0.18` (téléchargement + déchiffrement média),
`ruma-events 0.34` (blocs MSC3245 `audio` / `voice`), `tokio`, `serde_json`, `tracing`.

---

## 4. Détection d'un message vocal

### 4.1 Source des données

Un vocal arrive comme `MessageType::Audio(AudioMessageEventContent)` (parfois
`MessageType::File` selon le bridge). `ruma-events 0.34` expose déjà
(`src/room/message/audio.rs`) :

```rust
pub struct AudioMessageEventContent {
    pub body: String,
    pub filename: Option<String>,
    pub source: MediaSource,          // Plain(uri) | Encrypted(file)
    pub info: Option<Box<AudioInfo>>, // mimetype, size, duration
    pub audio: Option<UnstableAudioDetailsContentBlock>, // MSC3245 : duration + waveform
    pub voice: Option<UnstableVoiceContentBlock>,        // MSC3245 : marqueur "c'est un vocal"
    ...
}
```

### 4.2 Algorithme de classification

Fonction pure à ajouter dans un nouveau module `src/transcribe/detect.rs` :

```rust
pub enum AudioKind {
    /// Note vocale avérée (marqueur explicite).
    Voice,
    /// Probablement une note vocale (heuristique).
    ProbablyVoice,
    /// Fichier audio quelconque.
    Audio,
    /// Pas un audio du tout.
    NotAudio,
}

pub fn classify(msgtype: &MessageType) -> AudioKind;
```

Règles, dans l'ordre (premier match gagne) :

1. **`Voice`** — `MessageType::Audio` **et** `content.voice.is_some()`
   (bloc MSC3245 `org.matrix.msc3245.voice.v2`). C'est le cas d'Element, de
   mautrix-whatsapp et de mautrix-signal récents : critère de référence.
2. **`Voice`** — présence du bloc `content.audio` (MSC1767 audio details) **avec** une
   `waveform` non vide : les bridges qui n'émettent pas `voice` émettent souvent la waveform.
3. **`ProbablyVoice`** — `MessageType::Audio` et **toutes** les conditions :
   - `mimetype` ∈ { `audio/ogg`, `audio/ogg; codecs=opus`, `audio/opus`,
     `audio/mp4`, `audio/m4a`, `audio/aac`, `audio/amr`, `audio/webm` } ;
   - `duration` absente ou ≤ `max_duration` (défaut 15 min) ;
   - le nom de fichier matche une heuristique de vocal :
     extension ∈ { `.ogg`, `.oga`, `.opus`, `.m4a`, `.aac`, `.amr`, `.webm` }
     **ou** le nom matche `(?i)^(voice|vocal|audio)[-_ ]?(message|note)?` ou
     `(?i)^PTT-\d{8}` (format WhatsApp « push to talk »).
4. **`ProbablyVoice`** — `MessageType::File` dont le `mimetype` est audio et dont
   l'extension est `.ogg`/`.opus` (certains vieux bridges envoient `m.file`).
5. **`Audio`** — tout autre `MessageType::Audio`.
6. **`NotAudio`** — le reste.

> Le cas WhatsApp demandé dans la demande initiale tombe en règle 1 (bridge récent) ou
> règle 3 (`.ogg` opus, nom `PTT-20260918-WA0003.ogg`).

Le seuil de déclenchement de la transcription automatique est configurable :
`auto = "off" | "voice" | "probably_voice" | "all_audio"` (cf. §6).

### 4.3 Tests de la détection

Tests unitaires purs (pas de réseau) dans `src/transcribe/detect.rs`, à partir de JSON
d'événements réels capturés :

- vocal Element (bloc `voice`) → `Voice`
- vocal mautrix-whatsapp (`PTT-*.ogg`, waveform) → `Voice`
- vocal mautrix-whatsapp ancien (ogg, pas de bloc) → `ProbablyVoice`
- morceau de musique `.mp3` envoyé en `m.audio` → `Audio`
- `.ogg` envoyé en `m.file` → `ProbablyVoice`
- image/texte → `NotAudio`

---

## 5. Architecture du pipeline

```
 scrollback (chat.rs)
        │  message sélectionné / auto-détection au rendu
        ▼
 TranscriptionManager  (src/transcribe/mod.rs — calqué sur PreviewManager)
        │  état par clé MediaSource::unique_key()
        │  Queued → Downloading → Transcribing → Done(text) | Error(msg)
        ▼
 Requester::transcribe()  →  WorkerTask::Transcribe(source, info, permits)
        │                      (src/worker.rs)
        ▼
 1. cache disque hit ?  ──oui──▶ Done
        │ non
        ▼
 2. media.get_media_content(&MediaRequestParameters{source, MediaFormat::File}, true)
        │  (déchiffre automatiquement les médias E2EE)
        ▼
 3. écriture dans {dirs.cache}/transcribe/<unique_key>.<ext>
        ▼
 3bis. TRANSCODAGE → PCM WAV 16 kHz mono  (§5.1 — étape obligatoire)
        ▼
 4. backend STT (trait SpeechToText)   ── timeout, retry ──▶ texte
        ▼
 5. écriture cache {dirs.cache}/transcribe/<unique_key>.json
        ▼
 6. store.lock().application.transcriptions.insert(key, Done(text))
        ▼
 rendu dans le scrollback au prochain draw
```

Points de conception :

- **Réutilise la pile média de matrix-sdk** : pas de HTTP manuel vers le homeserver, le SDK
  gère les MXC, le `MediaRetentionPolicy` et le déchiffrement E2EE.
- **Concurrence bornée** par un `Semaphore` (défaut 2 transcriptions simultanées), comme
  `PreviewManager::permits` (`src/preview.rs:50`).
- **Jamais bloquant pour l'UI** : tout passe par le worker, l'UI ne fait que lire un état.

### 5.1 Transcodage audio — étape obligatoire

> **Correction d'une erreur de la v1 de cette spec.** La v1 passait le fichier Matrix
> directement à `whisper-cli` via `-f %FILE%`. **Cela échoue systématiquement.**

Les moteurs de la famille whisper.cpp n'acceptent que du **PCM WAV 16 kHz mono 16 bits**.
Or les notes vocales Matrix/WhatsApp sont de l'**Opus dans un conteneur OGG**. Un transcodage
est donc requis avant toute inférence, quel que soit le backend :

```
ffmpeg -nostdin -loglevel error -i <input> -ar 16000 -ac 1 -c:a pcm_s16le <output>.wav
```

Conséquences :

- le transcodage est une **étape du pipeline** (3bis), pas un détail d'implémentation d'un
  backend : tous les backends locaux en dépendent ;
- il s'exécute dans le worker, sous le même `Semaphore` ;
- `ffmpeg` absent du `PATH` → erreur explicite `IambError::Transcode("ffmpeg introuvable")`,
  détectée **au démarrage** si un backend local est configuré, pas au premier vocal ;
- les backends distants type OpenAI acceptent l'ogg/opus nativement : le transcodage y est
  **sauté** (l'API accepte le fichier d'origine, ce qui évite une perte de qualité et du CPU) ;
- le `.wav` intermédiaire est temporaire et supprimé après inférence, indépendamment de
  `keep_audio` (qui ne concerne que l'audio source).

La voie « zéro dépendance externe » (décodage Opus en pur Rust) est discutée en §13.3.

---

## 6. Configuration

Nouvelle section dans `config.json`/`config.toml` (cf. `src/config.rs`, struct `Tunables` /
`TunableValues`) :

```toml
[settings.transcription]
enabled = true          # défaut: false
auto = "voice"          # off | voice | probably_voice | all_audio   (défaut: off)
backend = "openai"      # openai | local | http
language = "auto"       # hint ISO-639-1, "auto" = détection par le modèle
max_file_size = 26214400  # octets, défaut 25 MiB (limite API OpenAI)
max_duration = 900        # secondes, défaut 15 min
concurrency = 2
timeout = 120             # secondes par requête
cache = true              # persister les transcriptions sur disque
keep_audio = false        # supprimer le .ogg du cache après transcription
redact_in_logs = true     # ne jamais logger le texte transcrit

# backend = "openai" (ou tout endpoint compatible OpenAI : groq, local litellm, …)
[settings.transcription.openai]
endpoint = "https://api.openai.com/v1/audio/transcriptions"
model = "whisper-1"
api_key_command = ["pass", "show", "openai/api-key"]   # recommandé
# api_key = "sk-..."                                    # déconseillé (secret en clair)

# backend = "local" : binaire externe, l'audio TRANSCODÉ (wav 16k mono) est passé en fichier
# NB: chemin secondaire — recharge le modèle à chaque appel (cold start 1-3 s).
#     Préférer whisper-server + backend = "openai" sur localhost (cf. §6.1).
[settings.transcription.local]
command = ["whisper-cli", "-m", "/models/ggml-small-q5_1.bin", "-otxt", "-nt", "-f", "%FILE%"]
# %FILE% est remplacé par le chemin du WAV 16 kHz mono ; stdout = transcription

# backend = "whisper-local" : inférence EMBARQUÉE dans iamb (feature Cargo, cf. §6.2)
[settings.transcription.whisper_local]
model = "small-q5_1"      # tiny | base | small | medium | large-v3 (+ variantes quantisées)
model_path = ""           # vide = {dirs.data}/models/, téléchargé au premier usage
threads = 0               # 0 = auto (nb de cœurs physiques)

# backend = "http" : POST multipart générique
[settings.transcription.http]
endpoint = "http://127.0.0.1:9000/asr"
field = "audio_file"
method = "POST"
headers = { "X-Api-Key" = "..." }
response_path = "text"   # chemin JSON du texte ; vide = corps brut en texte
```

Règles de validation au chargement :

- `enabled = true` avec `backend = "openai"` sans clé ni `api_key_command` → erreur de
  config explicite au démarrage (pas un échec silencieux à la première transcription).
- `api_key_command` est exécuté **une fois au démarrage** ; le secret n'est jamais écrit
  sur disque ni dans les logs.
- `auto != "off"` doit afficher un avertissement au premier lancement (cf. §9).

### 6.1 `backend = "openai"` sur localhost = backend local sans code supplémentaire

`whisper.cpp` fournit `whisper-server`, qui expose un endpoint **compatible avec l'API
OpenAI** (`POST /v1/audio/transcriptions`, multipart, réponse JSON `{"text": ...}`) :

```
whisper-server -m models/ggml-small-q5_1.bin --host 127.0.0.1 --port 8080
```

```toml
[settings.transcription]
backend = "openai"
[settings.transcription.openai]
endpoint = "http://127.0.0.1:8080/v1/audio/transcriptions"
model = "whisper-1"     # ignoré par whisper-server
# aucune api_key requise
```

Conséquence d'architecture : **le backend « local » et le backend « OpenAI » sont le même
code HTTP, seule l'URL change.** Aucune gestion de subprocess, aucun parsing de stdout,
aucune impl supplémentaire.

C'est aussi le seul moyen d'éviter le défaut majeur du mode subprocess : `whisper-cli`
**recharge le modèle à chaque invocation** (1–3 s de cold start). Tolérable pour un
`:transcribe` manuel, absurde en mode `auto`. Un serveur persistant charge le modèle une fois.

Ce chemin ne nécessite **aucune modification du code d'iamb** au-delà du backend HTTP déjà
prévu : c'est pourquoi il est implémenté en premier (P3/P4) et sert de banc d'essai à tout le
reste du pipeline.

**Choix de modèle** — `base` est faible en français. Recommandation de départ :
**`small` quantisé q5_1** (~180 Mo), bon compromis qualité/vitesse sur CPU multicœur.
`medium`/`large-v3` si la qualité prime sur la latence.

> Estimation non mesurée : sur un CPU 16 cœurs récent, un vocal de 20 s avec `small-q5_1`
> devrait se transcrire en quelques secondes en CPU pur. À confirmer par un benchmark réel
> avant d'activer `auto`. L'accélération NPU/OpenVINO/Vulkan n'est pas nécessaire au départ
> et coûte cher en configuration.

### 6.2 Inférence embarquée dans iamb (feature Cargo `whisper-local`)

**Cible assumée de la spec**, pas une note de bas de page : pour un usage personnel, un
**binaire unique, sans daemon à lancer, sans `ffmpeg` dans le `PATH`, fonctionnel hors
ligne** est l'ergonomie correcte pour un client TUI.

Implémentation : `whisper-rs` (bindings Rust sur `whisper.cpp`), exposé comme **quatrième
impl du trait `SpeechToText`**. L'abstraction du §8.2 rend l'ajout mécanique.

```toml
[features]
default = ["bundled", "desktop"]      # INCHANGÉ — jamais activé par défaut
whisper-local = ["dep:whisper-rs"]    # opt-in explicite
```

Le précédent existe déjà dans le `Cargo.toml` du projet : `chafa-static` / `chafa-dyn`
suivent exactement ce motif (dépendance native lourde, hors `default`).

**Coûts réels, à assumer en connaissance de cause :**

| Coût | Portée |
|---|---|
| Toolchain **cmake + compilateur C++** requis à la compilation | uniquement avec `--features whisper-local` |
| Temps de compilation d'iamb nettement allongé | idem |
| Cross-compilation plus pénible | idem |
| Poids du modèle 180 Mo – 1,5 Go à télécharger au premier usage | **commun à toutes les approches locales**, ce n'est pas un argument contre l'embarqué |

**Ce qui ne s'embarque pas :** les poids. Trop volumineux pour le binaire, pour un paquet
distribution, et pour crates.io (limite 10 Mo). « Embarqué » signifie donc toujours
*téléchargement au premier lancement* vers `{dirs.data}/models/`, avec vérification de
checksum, reprise sur échec et message de progression explicite dans l'UI.

**Contexte de décision :** les objections liées à `cargo install`, au packaging Debian et à
la limite crates.io ne s'appliquent **pas à un fork personnel** qui n'est pas distribué en
amont. Elles ne seraient bloquantes que pour une contribution à l'iamb upstream. Pour un
fork, la seule contrainte qui subsiste est la présence du toolchain C++ sur la machine de
build — déjà probable, le projet compilant `bundled-sqlite` (code C).

**Ordre d'implémentation — la seule réserve maintenue.** Construire d'abord le trait
`SpeechToText` + le backend HTTP (§6.1), *ensuite* `whisper-local`. Non par prudence : parce
que le backend HTTP est trivial à brancher et permet de déboguer **tout le reste du pipeline**
(détection, download, déchiffrement E2EE, transcodage, rendu scrollback, cache, invalidation
des hauteurs) avant d'introduire la variable « est-ce que les bindings C++ linkent ». Si
`whisper-rs` pose problème, on veut savoir que le reste fonctionne déjà.

---

## 7. Interface utilisateur

### 7.1 Commandes

| Commande | Effet |
|---|---|
| `:transcribe` | Transcrit la pièce jointe audio du message sélectionné. Erreur `IambError::NoAudioAttachment` si le message n'a pas d'audio. |
| `:transcribe!` | Force : ignore le cache et les heuristiques (transcrit même un `Audio` non-vocal). |
| `:transcribe stop` | Annule la transcription en cours pour le message sélectionné. |
| `:transcribe copy` | Copie la transcription dans le registre / presse-papier (via le mécanisme de registre modalkit déjà utilisé). |

Implémentation : `fn iamb_transcribe(desc, ctx)` dans `src/commands.rs`, enregistrée par
`add_command` à côté de `download`/`open`, produisant
`IambAction::Message(MessageAction::Transcribe(TranscribeFlags))`.

### 7.2 Keybinding

Par défaut : aucun (respect de la philosophie vim d'iamb). Documenté comme exemple :

```toml
[macros."normal"]
"gt" = ":transcribe<Enter>"
```

### 7.3 Rendu dans le scrollback

Sous le corps du message, indenté, avec le style `Style::default().add_modifier(ITALIC).dim()` :

```
 12:04  Alice  [Attached Audio: PTT-20260918-WA0003.ogg (43.1 kB, 0:17)]
                 ⟳ transcription en cours…
```

puis :

```
 12:04  Alice  [Attached Audio: PTT-20260918-WA0003.ogg (43.1 kB, 0:17)]
                 « Salut, je te rappelle dans dix minutes, j'suis dans le
                   métro là. À toute. »
```

et en cas d'échec :

```
                 ⚠ transcription échouée : HTTP 413 (fichier trop volumineux)
```

Contraintes de rendu :

- le texte est **wrappé** à la largeur disponible comme le corps de message (réutiliser
  `TextPrinter` de `src/message/printer.rs`) ;
- il ne doit **pas** être inclus dans la sélection/copie du corps original par défaut ;
- il compte dans la hauteur du message → invalider les hauteurs mises en cache quand une
  transcription passe à `Done` (point d'attention : `src/message/state.rs`).

---

## 8. Détail technique par fichier

### 8.1 Nouveaux fichiers

```
src/transcribe/mod.rs      TranscriptionManager, TranscriptionStatus, orchestration
src/transcribe/detect.rs   classify() + tests unitaires (§4)
src/transcribe/transcode.rs  to_wav16k() via ffmpeg + probe_ffmpeg() (§5.1)
src/transcribe/backend.rs  trait SpeechToText + impls openai / local / http
src/transcribe/whisper.rs  WhisperRsBackend — #[cfg(feature = "whisper-local")] (§6.2)
src/transcribe/cache.rs    lecture/écriture du cache disque
```

### 8.2 API interne

```rust
// src/transcribe/mod.rs
pub enum TranscriptionStatus {
    Queued,
    Downloading,
    Transcribing,
    Done { text: String, language: Option<String>, backend: String },
    Error(String),
}

pub struct TranscriptionManager {
    permits: Arc<Semaphore>,
    /// Indexé par MediaSource::unique_key(), comme PreviewManager.
    entries: HashMap<String, TranscriptionStatus>,
}

impl TranscriptionManager {
    pub fn new(settings: &ApplicationSettings) -> Self;
    pub fn get(&self, source: &MediaSource) -> Option<&TranscriptionStatus>;
    pub fn register(&mut self, source: &MediaSource);      // → Queued
    pub fn load(&mut self, source: &MediaSource, info: AudioMeta, worker: &Requester);
    pub fn insert(&mut self, key: String, status: TranscriptionStatus);
}
```

```rust
// src/transcribe/backend.rs
#[async_trait]
pub trait SpeechToText: Send + Sync {
    async fn transcribe(&self, audio: &Path, meta: &AudioMeta)
        -> Result<Transcript, TranscribeError>;
    fn name(&self) -> &'static str;
}

pub struct Transcript { pub text: String, pub language: Option<String> }
```

Implémentations :

- `OpenAiBackend` — `multipart/form-data` (`file`, `model`, `language?`,
  `response_format=json`) vers `endpoint`, header `Authorization: Bearer <key>`.
  Utiliser le `reqwest` déjà tiré transitivement par `matrix-sdk` (à exposer comme
  dépendance directe avec les features `json` + `multipart`, en réutilisant le réglage
  `ssl_verify` des tunables).
- `LocalBackend` — `tokio::process::Command`, `%FILE%` substitué (WAV transcodé, cf. §5.1),
  stdout capturé, timeout via `tokio::time::timeout`, code de retour ≠ 0 → erreur avec les
  500 premiers caractères de stderr. **Chemin secondaire** : préférer §6.1.
- `WhisperRsBackend` — **compilé uniquement avec `--features whisper-local`** (§6.2).
  Inférence in-process via `whisper-rs` ; charge le modèle **une fois** dans un
  `OnceCell`/`Arc` partagé par le worker (jamais par appel) ; prend le WAV 16 kHz mono en
  `Vec<f32>` ; `n_threads` = cœurs physiques par défaut. Gère le téléchargement du modèle
  vers `{dirs.data}/models/` au premier usage, avec checksum et état de progression exposé
  dans l'UI (réutiliser `TranscriptionStatus::Downloading`).
- `HttpBackend` — POST multipart générique + extraction via `response_path` (JSON pointer
  simplifié, `serde_json`).

Un backend est sélectionné au démarrage et stocké en `Arc<dyn SpeechToText>` ; l'ajout d'une
impl ne touche à aucun autre module.

### 8.2.1 Transcodage

```rust
// src/transcribe/transcode.rs
/// Convertit un média audio quelconque en PCM WAV 16 kHz mono 16 bits (cf. §5.1).
pub async fn to_wav16k(input: &Path, output: &Path) -> Result<(), TranscodeError>;

/// Vérifie la disponibilité de ffmpeg. Appelé AU DÉMARRAGE si un backend local est actif.
pub fn probe_ffmpeg() -> Result<(), TranscodeError>;
```

### 8.3 Worker

```rust
// src/worker.rs
WorkerTask::Transcribe(MediaSource, AudioMeta, Arc<Semaphore>)
```

- ajouter le bras dans `impl Debug for WorkerTask` (ne **pas** logger le texte transcrit) ;
- `Requester::transcribe(&self, source, meta, permits)` envoie la tâche ;
- handler asynchrone `transcribe_media(store, media, source, meta, permits, settings)` calqué
  sur `preview::load_image` (`src/preview.rs:126`) : il écrit le résultat final dans
  `store.lock().await.application.transcriptions`.

### 8.4 Modifications

| Fichier | Modification |
|---|---|
| `src/base.rs` | `MessageAction::Transcribe(TranscribeFlags)` ; `bitflags TranscribeFlags { NONE, FORCE, STOP, COPY }` ; `ProgramStore.application.transcriptions: TranscriptionManager` ; nouvelles variantes `IambError::{NoAudioAttachment, TranscribeDisabled, Transcribe(String), AudioTooLarge(u64), AudioTooLong(u64)}` |
| `src/commands.rs` | `iamb_transcribe` + enregistrement de la commande + complétion |
| `src/windows/room/chat.rs` | bras `MessageAction::Transcribe` ; **factoriser** la logique de téléchargement de `MessageAction::Download` (chat.rs:156-221) en `async fn fetch_media(client, source) -> Result<Vec<u8>>` partagée |
| `src/message/mod.rs` | rendu de la transcription sous le message ; helper `Message::audio_source()` |
| `src/message/printer.rs` | wrapping du bloc transcription |
| `src/config.rs` | structs `Transcription` / `TranscriptionValues` + merge + validation ; `DirectoryValues::cache/transcribe` créé par `create_dir_all` (config.rs:1126) |
| `src/main.rs` | initialisation du backend STT au démarrage (résolution de `api_key_command`) |
| `docs/iamb.5` | documentation de `[settings.transcription]` |
| `docs/iamb.1` | documentation de `:transcribe` |
| `config.example.toml` | exemple commenté, désactivé |
| `README.md` | mention de la fonctionnalité + avertissement vie privée |

---

## 9. Vie privée et sécurité — **contraintes bloquantes**

iamb est un client **chiffré de bout en bout**. Envoyer un vocal à une API tierce casse cette
propriété pour ce média. Exigences non négociables :

1. `enabled = false` par défaut, `auto = "off"` par défaut. Aucun octet ne sort sans une
   action explicite de l'utilisateur.
2. À la **première** utilisation d'un backend distant, afficher une confirmation bloquante
   (`UIError::NeedConfirm` + `PromptYesNo`, mécanisme déjà utilisé en `chat.rs:150`) :
   « La transcription enverra ce média à `<endpoint>`. Le chiffrement de bout en bout ne
   protège plus ce contenu une fois transmis. Continuer ? (y/n) ». Mémoriser le
   consentement dans le répertoire `data`.
3. Aucune transcription automatique dans les rooms marquées comme sensibles si iamb expose
   un jour un tel marqueur ; en attendant, `auto` doit pouvoir être surchargé **par profil**.
4. `redact_in_logs = true` par défaut : ni le texte, ni la clé d'API, ni l'URL signée ne
   doivent apparaître dans les logs `tracing` (attention aux dérivations `Debug`).
5. Le cache disque est écrit avec les permissions `0600` (Unix) dans le répertoire `cache`
   du profil ; `keep_audio = false` supprime le fichier audio après transcription.
6. La clé d'API doit pouvoir venir d'une commande externe (`api_key_command`) pour éviter le
   secret en clair dans la config — cohérent avec le reste de l'écosystème.

---

## 10. Gestion des erreurs

| Cas | Comportement |
|---|---|
| Transcription désactivée | `IambError::TranscribeDisabled`, message d'aide pointant vers `[settings.transcription]` |
| Message sans audio | `IambError::NoAudioAttachment` |
| Fichier > `max_file_size` | `AudioTooLarge`, aucun appel réseau |
| Durée > `max_duration` | `AudioTooLong`, aucun appel réseau |
| Échec de download média (clé E2EE manquante) | remonter l'erreur matrix-sdk telle quelle |
| Timeout backend | `Error("timeout après Ns")`, état conservé, `:transcribe!` relance |
| HTTP 429 | 1 retry avec backoff exponentiel (2s, 8s), puis erreur |
| HTTP 4xx/5xx | erreur avec code + `message` du corps JSON si présent |
| Binaire local absent | erreur explicite « commande `X` introuvable » |
| `ffmpeg` absent alors qu'un backend local est configuré | `IambError::Transcode`, détecté **au démarrage** (§5.1), pas au premier vocal |
| Échec du transcodage (fichier corrompu, codec inconnu) | `Transcode` + stderr ffmpeg tronqué |
| Modèle whisper-local absent / checksum invalide | proposer le téléchargement ; en cas d'échec, erreur explicite avec le chemin attendu |

Les erreurs sont **par message** : elles n'invalident jamais l'état global ni le scrollback.

---

## 11. Plan d'implémentation

| Phase | Contenu | Livrable vérifiable |
|---|---|---|
| **P0** | `src/transcribe/detect.rs` + tests unitaires | `cargo test transcribe::detect` vert, zéro impact runtime |
| **P1** | Config `[settings.transcription]` + validation + doc | `cargo test config`, démarrage avec/sans section |
| **P2** | Factorisation `fetch_media` depuis `chat.rs:156` | `:download` / `:open` inchangés (non-régression) |
| **P3** | `TranscriptionManager` + `WorkerTask::Transcribe` + transcodage ffmpeg (§5.1) + backend HTTP | `:transcribe` fonctionne contre `whisper-server` local (§6.1) |
| **P4** | Backend `openai` distant + prompt de consentement + backend `local` subprocess | transcription via API, confirmation affichée une fois |
| **P5** | Rendu scrollback + invalidation des hauteurs | affichage correct au redimensionnement |
| **P6** | Cache disque + `auto` + purge `keep_audio` | relance d'iamb : transcription restituée sans réseau |
| **P7** | Docs (`iamb.1`, `iamb.5`, `config.example.toml`, README) | `man` à jour |
| **P8** | Feature `whisper-local` : `WhisperRsBackend` + téléchargement de modèle (§6.2) | `cargo build --features whisper-local` puis `:transcribe` hors ligne, sans daemon ni ffmpeg |

P0→P2 sont sans risque et mergeables indépendamment. **P3 avec `whisper-server` valide tout
le pipeline avant que la moindre dépendance C++ n'entre dans le build** (§6.2, dernier §).

---

## 12. Tests

**Unitaires**

- `detect::classify` sur un corpus d'événements JSON (§4.3).
- Parsing/merge de la config, y compris valeurs invalides.
- Extraction `response_path` du backend `http`.
- Substitution `%FILE%` et échappement du backend `local`.

**Intégration (sans réseau)**

- Backend `local` avec un faux binaire (`sh -c 'echo bonjour'`) → `Done("bonjour")`.
- Backend `http` contre un serveur mock local (`tokio` + petit listener, ou `wiremock`).
- Cycle de cache : première passe appelle le backend, seconde non.
- Semaphore : 5 demandes simultanées, `concurrency = 2` → jamais plus de 2 en vol.

**Manuel**

- Vocal WhatsApp via mautrix-whatsapp dans une room E2EE.
- Vocal Element natif.
- Fichier `.mp3` de 60 min → refus `AudioTooLong` sans appel réseau.
- `:transcribe` sur un message texte → erreur propre.

---

## 13. Hors périmètre (itérations futures)

- Traduction de la transcription.
- Diarisation (identification des locuteurs).
- Transcription de la **vidéo** (`MessageType::Video`) — l'architecture le permet, il
  suffira d'ajouter l'extraction de piste audio via `ffmpeg` (§5.1 est déjà en place).
- Transcription **sortante** (dicter un message vocal depuis iamb).
- Résumé automatique des vocaux longs par LLM.
- Transcription temps réel / streaming.
- Accélération matérielle (NPU / OpenVINO / Vulkan / CoreML) pour `whisper-local`.

### 13.3 Décodage audio en pur Rust — voie « zéro dépendance externe »

Supprimer `ffmpeg` du pipeline (§5.1) rendrait la voie `whisper-local` (§6.2) réellement
auto-suffisante : un binaire, aucun exécutable tiers.

**Maillon faible identifié : le décodage Opus.** `symphonia` couvre Ogg/Vorbis, mais son
support **Opus** a longtemps été incomplet — état actuel non vérifié. Or Opus est
précisément le codec des notes vocales WhatsApp/Matrix, soit le cas d'usage central.

Conséquence logique : si Opus impose de passer par `libopus` en FFI, l'argument
« embarqué = plus aucune dépendance native » tombe, et `ffmpeg` (déjà présent, déjà éprouvé)
reste le choix raisonnable. **À vérifier avant de s'engager sur cette voie** ; c'est la seule
inconnue technique bloquante restante du dossier.

Le rééchantillonnage 48 kHz → 16 kHz, lui, est trivial en Rust (`rubato` ou implémentation
directe) et ne pose pas de problème.

---

## 14. Questions ouvertes

1. Exposer `reqwest` en dépendance directe ou passer par le client HTTP interne de
   matrix-sdk ? (préférence : `reqwest` direct, features `json` + `multipart`, afin de ne
   pas dépendre d'internals non publics du SDK).
2. Le cache doit-il vivre sous `dirs.cache` (purgeable) ou `dirs.data` (persistant) ?
   Proposition : `dirs.cache/transcribe/`, cohérent avec le `MediaRetentionPolicy` existant.
3. Faut-il une transcription automatique bornée (« seulement les N derniers vocaux de la
   room visible ») pour éviter une avalanche d'appels API à l'ouverture d'un backlog ?
   Proposition : oui, `auto` ne déclenche que sur les messages **rendus à l'écran** et
   reçus depuis le démarrage de la session.
4. **Bloquante pour §13.3** : quel est l'état réel du support **Opus** dans `symphonia`
   aujourd'hui ? De la réponse dépend la possibilité d'une voie 100 % Rust sans `ffmpeg`.
5. Benchmark à mesurer (et non estimer) : latence de `small-q5_1` sur un vocal de 20 s, en
   CPU pur, sur la machine cible. Conditionne l'activation raisonnable de `auto`.

---

## Annexe A — Historique des révisions

**v2** — corrections issues de la revue :

1. **Bug corrigé (§5.1)** : la v1 passait le fichier `.ogg` directement à `whisper-cli`, ce
   qui échoue toujours (whisper.cpp n'accepte que du PCM WAV 16 kHz mono). Le transcodage
   `ffmpeg` devient une étape explicite du pipeline.
2. **§6.1 ajouté** : `whisper-server` expose une API compatible OpenAI ⇒ le backend local et
   le backend distant partagent le même code HTTP. Supprime une implémentation entière et
   évite le rechargement du modèle à chaque appel.
3. **§6.2 ajouté** : l'inférence embarquée via `whisper-rs` derrière la feature Cargo
   `whisper-local` est promue en **cible assumée**. Les objections liées à `cargo install`,
   au packaging et à crates.io ne valent que pour une contribution upstream, pas pour ce
   fork ; elles ont été retirées de l'argumentaire. Seule réserve maintenue : l'**ordre**
   d'implémentation (HTTP d'abord, bindings C++ ensuite).
4. **§13.3 ajouté** : le décodage Opus en pur Rust est la vraie inconnue de la voie « zéro
   dépendance externe » — à vérifier avant tout engagement.
5. **Modèle** : recommandation `small-q5_1` plutôt que `base`, insuffisant en français.
   Les chiffres de performance sont explicitement signalés comme **estimés, non mesurés**.

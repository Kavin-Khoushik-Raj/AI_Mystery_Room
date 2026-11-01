# Decisions

- The first release is a working responsive client with an API-backed experience and an offline demo fallback. This lets the core game remain playable before MongoDB or Gemini credentials are configured.
- Demo cases are generated locally from deterministic fictional templates; they are clearly marked as demo content. Production case generation uses the server Gemini pipeline.
- Authentication uses a signed JWT cookie when the server is configured. Local demo play stores progress in the browser.
- Image generation is optional and has a polished placeholder fallback so case creation does not depend on image-model availability.
- Sharing demo cases uses browser-local case data; API sharing uses persisted case records.

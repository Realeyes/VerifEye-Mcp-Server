# VerifEye MCP Server

The VerifEye MCP Server is a remotely hosted [Model Context Protocol](https://modelcontextprotocol.io/) (MCP) server from [Realeyes](https://realeyes.ai/). It exposes the VerifEye Face API as tools an AI assistant can call: analyze a face for apparent age and gender plus momentary emotions and attention (Identify), onboard a face so the same person is identified later (Onboard), compare an image against one onboarded face (Compare), and manage the collections onboarded faces live in.

The server is hosted by Realeyes, so there is nothing to install or run. This repository holds its public listing metadata for MCP registries: [`server.json`](server.json).

Full documentation: **https://verifeye-docs.realeyes.ai/verifeye-apis/mcp-server/**

## Server URL

```
https://verifeye-mcp-server.realeyes.ai/mcp
```

Transport: Streamable HTTP. This is the server's only address. If a registry or directory lists a different one, do not connect to it, and [contact us](mailto:verifeye@realeyes.ai).

## Requirements

- A [VerifEye Developer Console](https://verifeye-console.realeyes.ai/) account.
- An MCP client that supports remote MCP servers with OAuth.

## Authentication

The server uses OAuth 2.1 (authorization code with PKCE). When you connect, a VerifEye sign-in and consent screen opens in your browser. You never put an API key into your MCP client: your account's API key is resolved on the server side and is never exposed to the client.

The server offers no dynamic client registration. ChatGPT, VS Code and Codex identify themselves with a Client ID Metadata Document, so there is nothing to enter. Claude, Claude Code, Cursor and the Gemini CLI use a pre-registered client ID.

## Connecting a client

Step-by-step setup for Claude Code, Claude, ChatGPT, VS Code, Cursor, the Gemini CLI and Codex, including the client ID each one needs, is in the documentation: [Connecting a client](https://verifeye-docs.realeyes.ai/verifeye-apis/mcp-server/#connecting-a-client).

For example, in VS Code add `.vscode/mcp.json` to your project root:

```json
{
  "servers": {
    "verifeye": { "type": "http", "url": "https://verifeye-mcp-server.realeyes.ai/mcp" }
  }
}
```

Start the server from the **MCP: List Servers** command, then sign in when the browser opens.

## Using the server

Once the server is connected and you have signed in, ask in plain language and the assistant picks the tools.

**Give the image as a link.** The image tools take a public URL of a JPEG or PNG image, which VerifEye fetches itself. An image attached to the conversation cannot be passed to a tool, so put the image somewhere reachable and give the assistant that link. Prefer a pre-signed link that expires in minutes to a durably public URL. Where the server offers `verifeye_get_upload_url` and your client can make HTTPS requests of its own, the client can upload the image instead. Image requirements: [documentation](https://verifeye-docs.realeyes.ai/verifeye-apis/image-requirements/).

**Choose the region.** Every call runs in the EU (`eu`) unless you ask for the US (`us`). Collections are per region, so ask for the same region each time you work with a collection.

### Example prompts

The image links are placeholders: use a link to your own image. Prompts 2 to 5 build on each other and use the collection and the face that prompt 2 creates.

| # | Prompt | What happens |
|---|--------|--------------|
| 1 | "Estimate the apparent age and gender of the face in https://example.com/photos/visitor.jpg." | `verifeye_identify_face` analyzes the face and returns the estimates. No face or image is stored. |
| 2 | "Create a VerifEye collection called front-desk-visitors, then onboard the face in https://example.com/photos/visitor-1042.jpg into it with the external key visitor-1042." | `verifeye_create_collection` creates the collection, and `verifeye_onboard_face` stores the face template with that key. Onboarding stores biometric data: see [Biometric data and consent](#biometric-data-and-consent). |
| 3 | "Is the person in https://example.com/photos/today.jpg already in my front-desk-visitors collection?" | `verifeye_identify_face` with that collection answers `Existing`, with the `faceId` and the key, when the photo shows the person onboarded in prompt 2, and `NotExisting` for anyone else. Nothing is onboarded. |
| 4 | "Compare the face in https://example.com/photos/check-in.jpg with visitor-1042 in front-desk-visitors." | `verifeye_get_face_by_key` finds the `faceId`, and `verifeye_compare_face` returns the similarity and VerifEye's `isMatch` verdict. |
| 5 | "Delete visitor-1042's face from front-desk-visitors." | `verifeye_get_face_by_key` finds the face, and `verifeye_delete_face` destroys its template. This cannot be undone. |

Many clients ask you to approve a call before it runs, for example before a face is onboarded or deleted.

## Tools

Every tool takes `region` (`eu` or `us`, `eu` by default), which selects where the image is processed and where a collection lives. Collections are per region: one created in `eu` is not visible from `us`.

| Tool | What it does | Stores or destroys data |
|------|--------------|-------------------------|
| `verifeye_identify_face` | Analyzes the face(s) in an image, and identifies them against a collection if you name one | No |
| `verifeye_onboard_face` | Onboards the face(s) in an image into a collection so the person is identified later | **Stores a face template** |
| `verifeye_compare_face` | Compares the face(s) in an image against one onboarded face | No |
| `verifeye_delete_face` | Deletes one onboarded face | **Irreversible** |
| `verifeye_list_collections` | Lists the collections on the account, or checks whether one exists | No |
| `verifeye_create_collection` | Creates an empty collection | No |
| `verifeye_set_collection_description` | Replaces a collection's description | No |
| `verifeye_reset_collection` | Deletes every face in a collection, keeping the collection | **Irreversible** |
| `verifeye_delete_collection` | Deletes a collection and every face in it | **Irreversible** |
| `verifeye_get_face_by_key` | Finds an onboarded face by your own identifier | No |
| `verifeye_set_face_key` | Attaches your own identifier to an onboarded face | No |
| `verifeye_delete_face_key` | Removes your identifier from an onboarded face | Loses the way of finding the face |
| `verifeye_get_upload_url` | Hands back a short-lived form for uploading one image | No |

Parameters, results and error codes for each tool are in the [documentation](https://verifeye-docs.realeyes.ai/verifeye-apis/mcp-server/#available-tools).

## Biometric data and consent

These tools process facial (biometric) data. You are responsible for obtaining valid, informed consent before a call runs, and for meeting the privacy and biometric-data regulations that apply to the people in your images (for example GDPR, BIPA and CCPA).

- `verifeye_onboard_face` stores a face template that persists until it is deleted.
- `verifeye_delete_face`, `verifeye_reset_collection` and `verifeye_delete_collection` destroy face templates permanently.
- Age is an apparent-age estimate, not a legal proof of age. Gender is a demographic estimate, not a self-declared identity attribute. A face match is evidence of similarity, not an identity check on its own.
- Tool calls run as your VerifEye account and count against your account's usage limits.

## Links

- Documentation: https://verifeye-docs.realeyes.ai/verifeye-apis/mcp-server/
- MCP Registry entry (`ai.realeyes/verifeye`): https://registry.modelcontextprotocol.io/v0.1/servers/ai.realeyes%2Fverifeye/versions/latest
- Privacy Policy: https://realeyes.ai/privacy-policy/
- Terms: https://realeyes.ai/terms-for-customers/
- Support: [verifeye@realeyes.ai](mailto:verifeye@realeyes.ai)

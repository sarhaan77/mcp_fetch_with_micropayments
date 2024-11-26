# Charge bots money to access your website!

I wanted to experiment with Anthropic's MCP and the /llms.txt standard to show how agents could pay small amounts of crypto to interact with websites that restrict access.

You can read more about it here: https://x.com/sarhaan77/status/1861266093064454390

This was an evening hack and works like any other MCP server, but it has a few extra features to support websites that want to charge bots money to access their content.

## Available Tools

- `fetch` - Fetches a URL from the internet and extracts its contents as markdown.
- `access` - Transfers a given amount of ETH to a recipient wallet address as specified in the /llms.txt of the website the user is trying to access.
- `proxy` - Fetches a URL from the internet through a proxy and extracts its contents as markdown. Called after the user has paid for access to a website.

## Installation

### Using uv (recommended)

When using [`uv`](https://docs.astral.sh/uv/) no specific installation is needed. We will
use [`uvx`](https://docs.astral.sh/uv/guides/tools/) to directly run _mcp-server-fetch_.

### Configure for Claude.app

Add to your Claude settings:

<details>
<summary>Using uvx</summary>

```json
{
  "mcpServers": {
    "fetch_with_micropayments": {
      "command": "uv",
      "args": [
        "--directory",
        "<path to fetch_with_micropayments repo>",
        "run",
        "mcp-server-fetch"
      ]
    }
  }
}
```

</details>

## Backend Infra and Website Configuration

### Proxy Server

This server is responsible for verifying the payment and sending the request to the website. It is whitelisted by the website so bot protection does not block it.

Right now our proxy server is extremely simple:

```python
from fastapi import FastAPI
from pydantic import BaseModel
import httpx
import uvicorn
from eth_account import Account
from eth_account.messages import encode_defunct

app = FastAPI()
client = httpx.AsyncClient(verify=False)


def verify_signature(message: str, signature: str, wallet_address: str):
    msg = encode_defunct(text=message)
    recovered_address = Account.recover_message(msg, signature=signature)
    return recovered_address.lower() == wallet_address.lower()


class UrlRequest(BaseModel):
    url: str
    signature: str
    wallet: str
    message: str


@app.post("/")
async def proxy(request: UrlRequest):
    is_valid = verify_signature(request.message, request.signature, request.wallet)
    if not is_valid:
        return ("Invalid signature",)

    response = await client.get(request.url)
    return response.text


if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

It is deployed as a digital ocean droplet with a static IP address and whitelisted by the website.
This is a POC so I did not bother adding checks for if the correct payment was made on chain, but it is trivial to add.

### Cloudflare setup

1. Ideally, you want to be on the pro plan. You can enable the bot protection feature, Block AI Bots, etc. BUT do not enable Bot Fight Mode as it block access to `/llms.txt` and cannot be bypassed via custom rules. You can enable Super Bot Fight Mode if you are on the pro plan.
2. Security/waf/custom rule: Add a rule to skip all rate limit/user agent/bot protection features if the URI Path is equal to `/llms.txt`.
3. Security/waf/custom rule: Add a rule to skip bot protection features if the IP Source Address is equal to the IP address of the proxy server.

### Setup testnet

1. I use [anvil](https://github.com/foundry-rs/foundry/tree/master/crates/anvil) to spin up a local testnet, expose it to the internet via `tailscale funnel`, and then use it to send and receive payments. You can use any testnet you like.

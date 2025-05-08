
# Connecting External Web Servers to Agents via MCP

## Purpose of This Work

Previously, I experimented with a basic local MCP server and also tested a remote MCP server provided by Smithery. This time, my goal is to directly connect an internal or external web server to an Agent via MCP. If successful, this would enable me to repurpose a variety of web servers into MCP servers for broader use.

## Background

The key task is to convert internal or external Webtoon-related API servers into MCP servers and connect them to Agents.

MCP server documentation is still sparse—especially for cases like converting a generic HTTP web server into an MCP server.

After some research, I found three primary approaches (most examples are built around FastAPI):

- **FastMCP** (FastAPI app compatible) → Most popular today  
- **FastAPI MCP** (designed for FastAPI servers)  
- **Manual conversion** (no libraries)

## Deliverables

Instead of using internal APIs, I first convert a public Webtoon API server into an MCP server. This is then integrated with LangChain and ChatGPT to create a basic *Conversational Webtoon Search Agent* with trivial queries like:

- “Can you recommend a work by Jo Seok?”  
- “Find Jo Seok’s most recent series and show me the best comment from episode 3.”

Along the way, I documented insights and lessons (including failed attempts) I gained during the process—with help from ChatGPT.

---

## Steps

### [1] Selecting and Preparing the External Webtoon API Server

I chose a public Webtoon API server: [https://github.com/HyeokjaeLee/korea-webtoon-api](https://github.com/HyeokjaeLee/korea-webtoon-api)

Swagger revealed the server is built with Node.js + Express.

Most MCP server libraries are geared toward FastAPI, so I planned to:

- Convert the Node.js API server into a **FastAPI proxy server**, then  
- Wrap it with an MCP server.

```python
from fastapi import FastAPI, Request
import httpx

fastapi_app = FastAPI()
EXISTING_API_BASE = "https://korea-webtoon-api-cc7dda2f0d77.herokuapp.com"

@fastapi_app.get("/webtoons")
async def proxy_webtoons(request: Request):
    async with httpx.AsyncClient() as client:
        r = await client.get(f"{EXISTING_API_BASE}/webtoons", params=request.query_params)
        return r.json()

@fastapi_app.get("/status")
async def proxy_status():
    async with httpx.AsyncClient() as client:
        r = await client.get(f"{EXISTING_API_BASE}/status")
        return r.json()
```

**Issue**: MCP couldn’t parse query params from this FastAPI proxy.  
This led me to explore libraries that can convert the server into an MCP-compliant server in one step.

---

### [2] Testing MCP Server Libraries

#### FastMCP  
[https://github.com/jlowin/fastmcp](https://github.com/jlowin/fastmcp)

The most popular option, and frequently used in LangChain examples.  
It offers simple conversion from a FastAPI app into an MCP server:

```python
from fastapi import FastAPI
from fastmcp import FastMCP

app = FastAPI()

@app.get("/status")
def get_status():
    return {"status": "running"}

@app.post("/items")
def create_item(name: str, price: float):
    return {"id": 1, "name": name, "price": price}

mcp_server = FastMCP.from_fastapi(app)

if __name__ == "__main__":
    mcp_server.run()
```

**Problem**: This only works if I’ve written the API as a FastAPI app. It’s not ideal when working with a third-party Node.js API that’s being proxied.

In FastMCP, GET methods are treated as `resources` rather than `tools`, and the documentation/examples are limited.

LangChain’s MCP adapter only recently added support for `resources`, and documentation is still lacking.

I attempted a workaround by treating GET methods as POST tools, but query params were still not handled properly.

---

#### FastAPI MCP  
[https://github.com/tadata-org/fastapi_mcp](https://github.com/tadata-org/fastapi_mcp)

Also covered in this Hugging Face blog: [https://huggingface.co/blog/lynn-mikami/fastapi-mcp-server](https://huggingface.co/blog/lynn-mikami/fastapi-mcp-server)

This library is tailored for FastAPI, making it suitable for my proxy setup.

**Key Benefits**:
- Can turn a FastAPI proxy into an MCP server in one step.  
- All endpoints (even GET) are treated as MCP tools.  
- Smooth integration with LangChain clients.

```python
from fastapi import FastAPI
from fastapi_mcp import FastApiMCP
import httpx

EXISTING_API_BASE = "https://korea-webtoon-api-cc7dda2f0d77.herokuapp.com"
app = FastAPI()

async def proxy_get_webtoons(keyword: str = None, genre: str = None):
    params = {}
    if keyword: params["keyword"] = keyword
    if genre: params["genre"] = genre

    async with httpx.AsyncClient() as client:
        response = await client.get(f"{EXISTING_API_BASE}/webtoons", params=params)
        return response.json()

app.add_api_route("/", proxy_get_webtoons, methods=["GET"], operation_id="get_webtoons")
mcp = FastApiMCP(app, name="My API MCP", description="MCP server for my API")

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8000)
```

---

### [3] LangChain + Client Integration Test

I successfully connected the MCP server to a LangChain client using SSE.  
Tool schemas were auto-parsed, allowing tool-calling via LLM:

```python
import asyncio
from langchain_mcp_adapters.client import MultiServerMCPClient
from langchain_mcp_adapters.tools import load_mcp_tools
from langgraph.prebuilt import create_react_agent
from langchain_openai import ChatOpenAI

model = ChatOpenAI(model="gpt-4o")

async def main():
    async with MultiServerMCPClient(
        {
            "webtoon": {
                "url": "http://0.0.0.0:8000/mcp",
                "transport": "sse",
            }
        }
    ) as client:
        tools = client.get_tools()
        agent = create_react_agent(model, tools)
        response = await agent.ainvoke({"messages": "Find a recent comic by Jo Seok"})
        print("Agent response:", response)

if __name__ == "__main__":
    asyncio.run(main())
```

---

### [4] Host Service Simulation

I extended the setup with a basic Streamlit UI to simulate a Host service platform.

I combined the local Webtoon MCP server with remote MCP tools like Playwright Automation.

**Sample query**: “Find Jo Seok’s latest series and show the best comment from episode 3.”

- **Step 1**: Webtoon MCP server retrieves Jo Seok's works.  
- **Step 2**: Remote Playwright MCP automates browser navigation and collects comments.  
- **Step 3**: ChatGPT extracts and summarizes the best comment.

---

## Conclusion

I successfully converted an external Web API into an MCP server and connected it to an Agent for a working proof of concept.

That said, MCP-based architectures are still maturing. Many components need to interoperate:

**Agent (LLM) ⇄ Framework (LangChain) ⇄ MCP ⇄ MCP Server Library**

Each of these is evolving rapidly, which causes occasional compatibility friction.

MCP supports not only `tools`, but also `resources`, `prompts`, etc.  
Still, I wonder:
- Will formats beyond `tool` gain widespread usage?
- If not, will support be dropped in future versions?

When tool responses include heavy data like entire webpages, token efficiency becomes critical. Preprocessing and summarization will be essential.

Going forward, I believe the most valuable direction lies not in technical exploration alone, but in identifying **high-upside, real-world business scenarios**.

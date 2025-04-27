# Creating an MCP Server Using Python and the Open-Meteo API

MCP servers extend the capabilities of language models by connecting them to data sources and services. Practically speaking, they are agnostic applications that facilitate integration with whatever data or service you point at and do pretty much anything you can think of. Think function calling, except the functions are plugins, and you start getting the idea.

## MCP Server Primitives

MCP servers expose three main primitives:

### Resources
*client-controlled*

Exposes data to be used as context to clients on request.

Use a resource when you want to passively expose data or content to an MCP client and let the client choose when to access it.

### Tools
*model-controlled*

Exposes executable functionality to client models.

Use a tool when you want the client to perform an action, for example to request content from an API, transform data, or send an email.

### Prompts
*user-controlled*

Exposes reusable prompts and workflows to users.

Use a prompt when you want the client to surface prompts and workflows to the user, for example to streamline interaction with a resource or tool.

## Let's talk about the weather

LLMs are great for transforming data into natural language. One practical example of this is how they can translate weather data like temperature, wind speed, dew point, etc into descriptions of what the weather will feel like and recommendations on what type of clothing to wear.

In this tutorial you'll build an MCP server that uses the Open-Meteo API to provide real-time weather information and weather forecasts. The Open-Meteo API is free for non-commercial use, easily configurable through query parameters, and does not require an API key which makes it ideal for LLM integration.

## How to use this tutorial

This tutorial shows you how to start from scratch and set up an MCP server locally on your computer. Keeping the files local makes it easier to test the MCP server in Claude Desktop.

As you follow along, use the fully built-out example in the open-meteo-weather folder in the exercise files repository for this course for reference.

## 0. Requirements

To follow along you need the following (much of this was covered earlier in the course):

- A Claude.ai account (MCP support is available for all account types)
- The Claude Desktop app, available for macOS and Windows
- A code editor like Visual Studio Code
- uv - a Rust-based Python package manager (full installation instructions):
    - macOS via Homebrew:
        ```
        brew install uv
        ```
    - Windows via WinGet:
        ```
        winget install --id=astral-sh.uv -e
        ```

## 1. Setting up the project

To set up your project, open your code editor to the folder you want to add your project. Then follow these steps to set up your project:

1. Create a new folder called mcp-server-weather using the editor tools or terminal:
     ```
     mkdir mcp-server-weather
     ```

2. Navigate to the folder in terminal:
     ```
     cd mcp-server-weather
     ```

3. Initiate a new uv project:
     ```
     uv init
     ```

4. Create a virtual environment using uv:
     ```
     uv venv
     ```

5. Start the virtual environment:
     ```
     source .venv/bin/activate
     ```
     Note: To stop the virtual environment, run `deactivate` in terminal.

6. Install the Python MCP SDK with the CLI extension and additional Python dependencies in the virtual environment:
     ```
     uv add "mcp[cli]" httpx
     ```

## 2. Building the weather MCP server

In the project folder there is a file called main.py. You can choose to work with this one, or create a new file called server.py. While the name is unimportant as long as you remember it for later, the naming convention for MCP servers is converging towards server.py so that's what this tutorial will use moving forward.

### Scaffolding

Start by adding scaffolding for the MCP server:

```python
from typing import Any
import httpx
from mcp.server.fastmcp import FastMCP

# Initialize FastMCP server
mcp = FastMCP("weather")

### The rest of your code goes between here...


### ... and here.

if __name__ == "__main__":
        # Initialize and run the server
        mcp.run(transport='stdio')
```

This adds strong typing, the httpx client for accessing the web, the FastMCP class for building MCP servers, and runs the MCP server using stdio (standard input/output) as the transport mechanism, which is what Claude Desktop and other MCP clients expect.

### Constants and helper functions

Next, add constants and a helper function to interact with the Open-Meteo API:

```python
# Constants
OPENMETEO_API_BASE = "https://api.open-meteo.com/v1"
USER_AGENT = "weather-app/1.0"

# Helper function to make a request to the Open-Meteo API
async def make_openmeteo_request(url: str) -> dict[str, Any] | None:
        """Make a request to the Open-Meteo API with proper error handling."""
        headers = {
                "User-Agent": USER_AGENT,
                "Accept": "application/json"
        }
        async with httpx.AsyncClient() as client:
                try:
                        response = await client.get(url, headers=headers, timeout=30.0)
                        response.raise_for_status()
                        return response.json()
                except Exception:
                        return None
```

### Create a get_forecast tool

The Open-Weather API has a /forecast endpoint that can provide current weather and hourly forecast for any location defined by latitude and longitude with optional parameters for the data you need. For the full specification, visit the API documentation.

Since we want the LLM to be able to automatically request data from the API, and may also want the MCP server to perform actions on that data, each API interaction in this MCP server is best exposed as a tool.

Here's how to build a tool requesting the current weather using the Python SDK:

```python
@mcp.tool()
async def get_current_weather(latitude: float, longitude: float) -> str:
        """Get current weather for a location.

        Args:
                latitude: Latitude of the location
                longitude: Longitude of the location
        """
        
        url = f"{OPENMETEO_API_BASE}/forecast?latitude={latitude}&longitude={longitude}&current=temperature_2m,is_day,showers,cloud_cover,wind_speed_10m,wind_direction_10m,pressure_msl,snowfall,precipitation,relative_humidity_2m,apparent_temperature,rain,weather_code,surface_pressure,wind_gusts_10m"
        
        data = await make_openmeteo_request(url)

        if not data:
                return "Unable to fetch current weather data for this location."

        return data
```

(See the official documentation for instructions on how to define tools without the SDK.)

- Tools are defined using `@mcp.tool()`
- The LLM uses the initial comment as its prompt for when and how to use the tool, so this is where you define the tool operation
- Tools are written as regular Python function with your specified arguments
- Tools should always return data
- The LLM receives the returned data for further processing

> **TIP:**
> Resist the urge to format the returned data! The tool above returns the entire data set unaltered to the LLM allowing the LLM to process the data and generate an appropriate answer. This runs counter to how we normally build software, but makes sense when we work with language models.

## Testing and running the MCP server

The MCP server is now fully functional and ready to test using the MCP Inspector. You'll learn more about the inspector in the next video, but here's a preview:

1. In terminal, start your MCP server in developer mode by running:
     ```
     mcp dev server.py
     ```
2. The MCP Inspector is now available at http://localhost:5173; open the URL in your browser
3. Select the "Connect" button
4. Select the "Tools" tab
5. Select the "List Tools" button
6. Select the get_current_weather tool
7. In the get_current_weather panel, enter a latitude and longitude, eg 63.4463991, 10.8127596
8. Under "Tool Result" you'll see a JSON object with weather data
9. Press Ctrl+C in terminal to crash out of the MCP Inspector.

## Extend the MCP server with more features

Now that you've built a tool, take what you've learned to add more tools and resources to the MCP server. Here are two ideas to get you started:

- `get_forecast` - Retrieves the forecast for the specified location, with an optional argument for the range of the forecast
- `get_location` - Uses the Open-Meteo Geocoding API for more accurate location searches (without this you are relying on the LLM to generate the latitude and longitude, which can result in errors)

## Troubleshooting

If your MCP server is not working as expected, compare it to the open-meteo-weather folder in the exercise files repository for this course, and watch the "Troubleshooting MCP servers" video later in this chapter. If you're still running into trouble, consult the official documentation for debugging tools and best-practices.
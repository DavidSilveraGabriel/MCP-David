# README.md para crear MCPs

## Introducción al MCP

El **Model Context Protocol (MCP)** es un protocolo abierto que estandariza la forma en que las aplicaciones proporcionan contexto a los Large Language Models (LLMs). Piensa en MCP como un puerto USB-C para aplicaciones de IA. Así como USB-C proporciona una forma estandarizada de conectar tus dispositivos a varios periféricos y accesorios, MCP proporciona una forma estandarizada de conectar modelos de IA a diferentes fuentes de datos y herramientas.

### ¿Por qué usar MCP?

MCP te ayuda a construir agentes y flujos de trabajo complejos sobre LLMs. Los LLMs frecuentemente necesitan integrarse con datos y herramientas, y MCP proporciona:

*   Una lista creciente de integraciones pre-construidas a las que tu LLM puede conectarse directamente.
*   La flexibilidad para cambiar entre proveedores y vendedores de LLM.
*   Mejores prácticas para asegurar tus datos dentro de tu infraestructura.

## Arquitectura General del MCP

En su núcleo, MCP sigue una arquitectura cliente-servidor donde una aplicación host puede conectarse a múltiples servidores:

```mermaid
graph TD
    A[Internet] --> B(MCP Protocol);
    C[Your Computer] --> B;
    B --> D(Host with MCP Client<br>(Claude, IDEs, Tools));
    D --> E(MCP Server A);
    D --> F(MCP Server B);
    D --> G(MCP Server C);
    E --> H(Local Data Source A);
    F --> I(Local Data Source B);
    G --> J(Remote Service C);
```

*   **MCP Hosts**: Programas como Claude Desktop, IDEs o herramientas de IA que desean acceder a datos a través de MCP.
*   **MCP Clients**: Clientes de protocolo que mantienen conexiones 1:1 con servidores.
*   **MCP Servers**: Programas ligeros que exponen capacidades específicas a través del Protocolo de Contexto del Modelo estandarizado.
*   **Local Data Sources**: Archivos, bases de datos y servicios de tu computadora a los que los servidores MCP pueden acceder de forma segura.
*   **Remote Services**: Sistemas externos disponibles a través de Internet (por ejemplo, a través de APIs) a los que los servidores MCP pueden conectarse.

## Conceptos Clave del MCP

### Servidor

El servidor FastMCP es tu interfaz principal con el protocolo MCP. Maneja la gestión de conexiones, el cumplimiento del protocolo y el enrutamiento de mensajes.

### Recursos

Los Recursos son la forma en que expones datos a los LLMs. Son similares a los endpoints GET en una API REST: proporcionan datos pero no deberían realizar cálculos significativos ni tener efectos secundarios.

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

@mcp.resource("config://app")
def get_config() -> str:
    """Static configuration data"""
    return "App configuration here"

@mcp.resource("users://{user_id}/profile")
def get_user_profile(user_id: str) -> str:
    """Dynamic user data"""
    return f"Profile data for user {user_id}"
```

### Herramientas (Tools)

Las Herramientas permiten a los LLMs realizar acciones a través de tu servidor. A diferencia de los recursos, se espera que las herramientas realicen cálculos y tengan efectos secundarios.

```python
import httpx
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

@mcp.tool()
def calculate_bmi(weight_kg: float, height_m: float) -> float:
    """Calculate BMI given weight in kg and height in meters"""
    return weight_kg / (height_m**2)

@mcp.tool()
async def fetch_weather(city: str) -> str:
    """Fetch current weather for a city"""
    async with httpx.AsyncClient() as client:
        response = await client.get(f"https://api.weather.com/{city}")
        return response.text
```

### Prompts

Los Prompts son plantillas reutilizables que ayudan a los LLMs a interactuar con tu servidor de manera efectiva.

```python
from mcp.server.fastmcp import FastMCP
from mcp.server.fastmcp.prompts import base

mcp = FastMCP("My App")

@mcp.prompt()
def review_code(code: str) -> str:
    return f"Please review this code:\n\n{code}"

@mcp.prompt()
def debug_error(error: str) -> list[base.Message]:
    return [
        base.UserMessage("I'm seeing this error:"),
        base.UserMessage(error),
        base.AssistantMessage("I'll help debug that. What have you tried so far?"),
    ]
```

### Imágenes

FastMCP proporciona una clase `Image` que maneja automáticamente los datos de imagen.

```python
from mcp.server.fastmcp import FastMCP, Image
from PIL import Image as PILImage

mcp = FastMCP("My App")

@mcp.tool()
def create_thumbnail(image_path: str) -> Image:
    """Create a thumbnail from an image"""
    img = PILImage.open(image_path)
    img.thumbnail((100, 100))
    return Image(data=img.tobytes(), format="png")
```

### Contexto

El objeto `Context` brinda a tus herramientas y recursos acceso a las capacidades del MCP.

```python
from mcp.server.fastmcp import FastMCP, Context

mcp = FastMCP("My App")

@mcp.tool()
async def long_task(files: list[str], ctx: Context) -> str:
    """Process multiple files with progress tracking"""
    for i, file in enumerate(files):
        ctx.info(f"Processing {file}")
        await ctx.report_progress(i, len(files))
        data, mime_type = await ctx.read_resource(f"file://{file}")
    return "Processing complete"
```

### Autenticación

La autenticación se puede utilizar en servidores que desean exponer herramientas que acceden a recursos protegidos. `mcp.server.auth` implementa una interfaz de servidor OAuth 2.0.

```python
from mcp import FastMCP
from mcp.server.auth.provider import OAuthAuthorizationServerProvider
from mcp.server.auth.settings import (
    AuthSettings,
    ClientRegistrationOptions,
    RevocationOptions,
)

class MyOAuthServerProvider(OAuthAuthorizationServerProvider):
    # See an example on how to implement at `examples/servers/simple-auth`
    ...

mcp = FastMCP(
    "My App",
    auth_server_provider=MyOAuthServerProvider(),
    auth=AuthSettings(
        issuer_url="https://myapp.com",
        revocation_options=RevocationOptions(
            enabled=True,
        ),
        client_registration_options=ClientRegistrationOptions(
            enabled=True,
            valid_scopes=["myscope", "myotherscope"],
            default_scopes=["myscope"],
        ),
        required_scopes=["myscope"],
    ),
)
```

## Guía Paso a Paso para Crear un MCP (con enfoque en Python SDK)

### Prerrequisitos

Asegúrate de tener Python instalado. Se recomienda usar `uv` o `pip` para gestionar las dependencias.

### Instalación del SDK

#### Añadir MCP a tu proyecto Python

Recomendamos usar [uv](https://docs.astral.sh/uv/) para gestionar tus proyectos Python.

Si aún no has creado un proyecto gestionado por uv, crea uno:

```bash
uv init mcp-server-demo
cd mcp-server-demo
```

Luego añade MCP a las dependencias de tu proyecto:

```bash
uv add "mcp[cli]"
```

Alternativamente, para proyectos que usan pip para las dependencias:

```bash
pip install "mcp[cli]"
```

#### Ejecutar las herramientas de desarrollo de MCP

Para ejecutar el comando `mcp` con uv:

```bash
uv run mcp
```

### Quickstart: Creando un Servidor Simple

Vamos a crear un servidor MCP simple que exponga una herramienta de calculadora y algunos datos:

```python
# server.py
from mcp.server.fastmcp import FastMCP

# Create an MCP server
mcp = FastMCP("Demo")

# Add an addition tool
@mcp.tool()
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b

# Add a dynamic greeting resource
@mcp.resource("greeting://{name}")
def get_greeting(name: str) -> str:
    """Get a personalized greeting"""
    return f"Hello, {name}!"
```

### Ejecutando tu Servidor

#### Modo Desarrollo

La forma más rápida de probar y depurar tu servidor es con el MCP Inspector:

```bash
mcp dev server.py
```

#### Integración con Claude Desktop

Una vez que tu servidor esté listo, instálalo en Claude Desktop:

```bash
mcp install server.py
```

#### Ejecución Directa

Para escenarios avanzados como despliegues personalizados:

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

if __name__ == "__main__":
    mcp.run()
```

Ejecútalo con:

```bash
python server.py
# o
mcp run server.py
```

#### Transporte HTTP Transmitible (Streamable HTTP Transport)

```python
from mcp.server.fastmcp import FastMCP

# Servidor con estado (mantiene el estado de la sesión)
mcp = FastMCP("StatefulServer")

# Servidor sin estado (sin persistencia de sesión)
mcp = FastMCP("StatelessServer", stateless_http=True)

# Ejecutar servidor con transporte streamable-http
mcp.run(transport="streamable-http")
```

Puedes montar múltiples servidores FastMCP en una aplicación FastAPI:

```python
# main.py
import contextlib
from fastapi import FastAPI
from mcp.echo import echo
from mcp.math import math

# Create a combined lifespan to manage both session managers
@contextlib.asynccontextmanager
async def lifespan(app: FastAPI):
    async with contextlib.AsyncExitStack() as stack:
        await stack.enter_async_context(echo.mcp.session_manager.run())
        await stack.enter_async_context(math.mcp.session_manager.run())
        yield

app = FastAPI(lifespan=lifespan)
app.mount("/echo", echo.mcp.streamable_http_app())
app.mount("/math", math.mcp.streamable_http_app())
```

#### Montaje en un Servidor ASGI Existente

Puedes montar el servidor SSE o Streamable HTTP en un servidor ASGI existente (como Starlette o FastAPI).

```python
from starlette.applications import Starlette
from starlette.routing import Mount
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("My App")

# Montar el servidor SSE en el servidor ASGI existente
app = Starlette(
    routes=[
        Mount('/', app=mcp.sse_app()),
    ]
)
```

## Mejores Prácticas

*   **Diseño Modular:** Separa la lógica en recursos y herramientas bien definidos.
*   **Uso Eficiente del Contexto:** Utiliza el objeto `Context` para acceder a las capacidades del MCP de manera efectiva.
*   **Seguridad:** Implementa autenticación para proteger recursos y herramientas sensibles.
*   **Pruebas y Depuración:** Utiliza el MCP Inspector y las herramientas de depuración proporcionadas por el SDK.

## Ejemplos Adicionales

El Python SDK incluye ejemplos más complejos que puedes explorar:

*   **Echo Server:** Un servidor simple que demuestra recursos, herramientas y prompts.
*   **SQLite Explorer:** Un ejemplo que muestra la integración con una base de datos SQLite.

Puedes encontrar estos ejemplos en el repositorio del [Python SDK](https://github.com/modelcontextprotocol/python-sdk/tree/main/examples/servers).

## Recursos Adicionales

*   [Documentación Oficial de MCP](https://modelcontextprotocol.io/)
*   [Especificación del Protocolo MCP](https://spec.modelcontextprotocol.io/)
*   [Repositorio del Python SDK](https://github.com/modelcontextprotocol/python-sdk)
*   [Servidores Oficialmente Soportados](https://github.com/modelcontextprotocol/servers)
*   [Guía de Contribución al Python SDK](https://github.com/modelcontextprotocol/python-sdk/blob/main/CONTRIBUTING.md)

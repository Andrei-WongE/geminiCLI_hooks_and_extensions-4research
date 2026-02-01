# Gemini CLI Hooks and Extensions for Research

A curated repository of Gemini CLI hooks and extensions designed for academic research, specifically focused on conceptual analysis and knowledge synthesis.

## Objective

As a PhD researcher, this project aims to leverage Gemini CLI hooks and extensions to connect conceptual ideas across a corpus of academic papers. The system integrates with Zotero (for paper management) and Notion (for note-taking) using the Model Context Protocol (MCP) to facilitate comprehensive literature analysis.

### Research Goals

The primary objective is to analyze a corpus of research papers to:

1. **Identify and connect conceptual ideas** across different papers
2. **Discover gaps** in the existing literature
3. **Find contradictions** between different studies or theories
4. **List and catalog definitions** used across the corpus
5. **Document methods** employed in the research literature

### Two-Phase Approach

#### Phase 1: Extract and Structure Concepts
- Extract key concepts from the research corpus
- Structure conceptual relationships
- Organize definitions and methodologies
- Create a knowledge graph of interconnected ideas

#### Phase 2: Ideate and Analyze
- Generate new conceptual links between ideas
- Identify research gaps and opportunities
- Highlight contradictions in the literature
- Suggest potential research directions

## Integration Points

- **Zotero**: Source of academic papers and references
- **Notion**: Repository for research notes and annotations
- **MCP (Model Context Protocol)**: Communication layer enabling AI-assisted analysis
- **Gemini CLI**: Command-line interface for executing research workflows

## Setup Guide: Deep Research Environment with Gemini CLI

This guide provides step-by-step instructions to configure a ""PhD-level"" research environment. We will install the **Co-Researcher** extension and integrate **Zotero** and **Notion** using a Model Context Protocol (MCP) to manage citations and knowledge bases.

### First approach
## Prerequisites on Windows 11
*   **Gemini CLI** installed and authenticated (`gemini auth login`).
*   **Zotero** desktop application running (for local API access).
*   **Notion** API Key (Internal Integration token).
*   **Python** (3.10+), uv pacakge and **Node.js** (LTS) installed.

## A. Install Co-Researcher Extension
First, we install the core research skills that provide methodology, peer review, and synthesis capabilities. 
Using slash commands like `/research` and `/literature-review` in CLI you will be able to activate those capabilities.

```bash PowerShell
# Install directly from GitHub
npx @google/gemini-cli extensions install https://github.com/poemswe/co-researcher --auto-update
```
## B. Initialize Zotero Semantic Search & RAG
I use **zotero-mcp** (by @54yyyu) to enable semantic search. This allows the agent to find papers based on concepts and meaning, not just keywords.

**Prerequisite:**
*   **Zotero 7+** must be running.
*   **Local API Enabled:** Go to Zotero → Settings → Advanced → Files and Folders → Check **"Allow other applications on this computer to communicate with Zotero"**.

** B.1 Index & Update Database
Before the agent can perform semantic searches, you must build the local vector database by indexing content.

**1. Initial Build (Full Text)**
Run this command in your PowerShell terminal to index your entire library, including the text inside your PDFs (required for deep analysis).

```bash PowerShell
# Using uv
uv tool run git+https://github.com/54yyyu/zotero-mcp.git update-db --fulltext
```

**2. Maintenance**
*   **Quick Update:** To update just metadata (fast) after adding new papers:
    ```bash PowerShell
    uv tool run git+https://github.com/54yyyu/zotero-mcp.git update-db
    ```
*   **Check Status:** To see if your database is ready:
    ```bash PowerShell
    uv tool run git+https://github.com/54yyyu/zotero-mcp.git db-status
    ```

** B.2. Connect and Configure
To connect this tool to Gemini CLI, add the server configuration to your `settings.json`. 

## C. Configure System Prompts & Hooks
To ensure consistent behavior, we use `GEMINI.md` for the persona and an **AfterTool hook** for citation handling.

** C.1. Create the System Prompt (`GEMINI.md`)
Save this file to `~/.gemini/GEMINI.md`. It defines the agent's role. Adjust content to your research project.

```markdown
# Research Assistant Protocol

Act as a research assistant with 10 years of social science methodology experience supporting a senior scientist.

## Primary Responsibilities
- Methodological rigor and appropriateness
- Data quality and integrity checks
- Identifying conflicting theories or contradictory evidence
- Critical evaluation of study designs and claims

## Approach
Maintain skeptical but constructive stance. Push back on weak evidence while acknowledging inherent uncertainty in social science research.
Consider alternatives. Use up-to-date info. Cite/verify citations. No exaggeration.

Protocol:
- Respond directly. No filler.
- Concise. Short sentences.
- Active voice. No apologies.

## Literature Review Standards

When reviewing literature:
- Flag methodological limitations
- Note sample sizes, populations, and generalizability issues
- Identify confounds or alternative explanations
- Highlight contradictions between studies
- Question causal claims from correlational data

# Format
**Key findings**
**Methodological strengths/weaknesses**
**Contradictions**

# Output Style
- Tables for comparisons, make it easily exportable.
- Confidence scores (0-1) per link.

# Study Hierarchy
- Clearly distinguish meta-analyses/systematic reviews from individual studies
- Prioritize meta-analyses as evidence base
- Use meta-analyses to identify key individual studies for deeper review

## Citation Rules
- **BibTeX Citation Keys**: Always use BibTeX Citation keys (e.g., `[@Smith2023]`) for citations, allowing easy citation in word.
- **Verification**: Before citing, verify the key exists in the Zotero library.
- Cite page numbers from PDFs.
- Use APA 7th edition for all citations
- For paywalled papers: Provide DOI and title so user can access through institutional login
```

** C.2. Create a Citation Hook
Save this script to `~/.gemini/hooks/format_citations.js`. It formats raw Zotero data into clean citations.

```java
#!/usr/bin/env node

// Read input from stdin
let inputData = '';
process.stdin.setEncoding('utf8');

process.stdin.on('data', (chunk) => {
  inputData += chunk;
});

process.stdin.on('end', async () => {
  try {
    const input = JSON.parse(inputData);
    
    // Log to stderr ONLY (never stdout)
    console.error('Hook triggered for tool:', input.tool_name);
    console.error('Tool input:', JSON.stringify(input.tool_input || {}));
    
    // Get the tool response
    let result = input.tool_response || {};
    
    // Check if this is a Zotero tool
    if (!input.tool_name || !input.tool_name.includes('zotero')) {
      console.error('Not a Zotero tool, skipping');
      process.stdout.write(JSON.stringify({ decision: "allow" }));
      process.exit(0);
    }
    
    let enrichmentCount = 0;
    
    // Extract/Generate Citation Keys
    if (result.items && Array.isArray(result.items)) {
      const MAX_ITEMS = 50; // Process up to 50 items
      const items = result.items.slice(0, MAX_ITEMS);
      
      console.error(`Processing ${items.length} items for citation keys`);
      
      for (const item of items) {
        let citationKey = null;
        
        // Method 1: Check item.data.extra field (Better BibTeX)
        if (item.data?.extra) {
          const citationMatch = item.data.extra.match(/Citation Key:\s*(\S+)/i) ||
                               item.data.extra.match(/bibtex:\s*(\S+)/i);
          if (citationMatch) {
            citationKey = citationMatch[1];
          }
        }
        
        // Method 2: Check if citationKey already exists
        if (!citationKey && item.citationKey) {
          citationKey = item.citationKey;
        }
        
        // Method 3: Check item.data.citationKey
        if (!citationKey && item.data?.citationKey) {
          citationKey = item.data.citationKey;
        }
        
        // Method 4: Generate from author + year
        if (!citationKey && item.data) {
          const creators = item.data.creators || [];
          const year = item.data.date ? item.data.date.match(/\d{4}/)?.[0] : '';
          
          if (creators.length > 0 && creators[0].lastName) {
            const lastName = creators[0].lastName.replace(/[^a-zA-Z]/g, '');
            citationKey = year ? `${lastName}${year}` : lastName;
          }
        }
        
        // Method 5: Fallback to item key
        if (!citationKey) {
          citationKey = `@${item.key}`;
        }
        
        // Add citation key to item
        item.citationKey = citationKey;
        enrichmentCount++;
        console.error(`Item ${item.key}: citation key = ${citationKey}`);
      }
      
      console.error(`✓ Enriched ${enrichmentCount} items with citation keys`);
    } else {
      console.error('No items found in response to enrich');
    }
    
    // Output result with minimal overhead
    const output = {
      decision: "allow",
      hookSpecificOutput: {
        additionalContext: enrichmentCount > 0 ? 
          `\n🔖 Citation keys added to ${enrichmentCount} Zotero items\n` : 
          ''
      }
    };
    
    // CRITICAL: Only JSON to stdout
    process.stdout.write(JSON.stringify(output));
    process.exit(0);
    
  } catch (error) {
    console.error('Hook error:', error.message);
    console.error('Stack:', error.stack);
    
    // Return valid JSON even on error
    process.stdout.write(JSON.stringify({
      decision: "allow",
      systemMessage: `Hook error: ${error.message}`
    }));
    process.exit(0);
  }
});

process.stdin.resume();
```
#### D. Setup Notion MCP and Local Notion Server
Instead of using a remote package, we will run the Notion MCP server locally for better control.

1.  **Clone the Server Repository**:
    ```bash PowerShell
    git clone https://github.com/modelcontextprotocol/servers.git mcp-servers
    cd mcp-servers/src/notion
    ```
2.  **Install Dependencies**:
    ```bash PowerShell
    npm install
    # Build the server
    npm run build
    ```
3.  **Get Credentials**:
    *   Create a "New Integration" at [Notion My Integrations](https://www.notion.so/my-integrations).
    *   Copy the **Internal Integration Secret**.
    *   **Action**: Go to your Notion project page, click `...` > `Connections`, and add your new integration.
    *   Configure Environment Variables `$env:NOTION_TOKEN="[secret_key]"`

4. **Install Notion MCP**
    * At the momnent (01/2026) the official Notion MCP does not allow for grannular access to pages and databases. [Possible BREAK POINT]
    * I used **SAhmadUmass/notion-mcp-server** from @SAhmadUmass to provide search, database querying, and page management capabilities.

**D.1. Build**
```bash
git clone https://github.com/SAhmadUmass/notion-mcp-server.git
cd notion-mcp-server
npm install
npm run build
```

### E. Complete Configuration File
Replace your existing `settings.json` (typically located at `%USERPROFILE%\.gemini\settings.json`) with the content below.

**Important:** Replace `[KEY]`, `[USER]` AND `[VERSION]` with your actual information. Verify PATHs!

```json
{
  "mcp": {
    "trust": true
  },
  "mcpServers": {
    "notion-enhanced": {
      "command": "node",
      "args": [
        "C:\\Users\\[USER]\\notion-mcp-server\\dist\\index.js"
      ],
      "env": {
        "NOTION_API_KEY": "ntn_[KEY]"
      }
    },
    "zotero": {
      "command": "uv",
      "args": [
        "tool",
        "run",
        "zotero-mcp",
        "serve"
      ],
      "env": {
        "ZOTERO_LOCAL": "true"
      }
    }
  },
  "security": {
    "auth": {
      "selectedType": "oauth-personal"
    }
  },
  "hooks": {
    "enabled": ["zotero-enrich"],
    "AfterTool": [
      {
        "matcher": "^(zotero_search_items|zotero_semantic_search|zotero_advanced_search)$",
        "sequential": false,
        "hooks": [
          {
            "name": "zotero-enrich",
            "type": "command",
            "command": "node C:\\Users\\[USER]\\.gemini\\hooks\\format_citations.js",
            "description": "Add Zotero citation keys to Zotero output",
            "timeout": 5000
          }
        ]
      }
    ]
  }
}
```
### 4. Verification
Restart your terminal and run the following commands to confirm the setup is active:

```bash Gemini CLI
# 1. Check Extensions and MCP servers
/extensions list
/mcp list
# Output: list of extensions and state of MPC servers, including GEMINI.md

# 2. Test Zotero & Notion Integration
gemini chat
> Check my Zotero and Notion for papers on 'Spatial Segregation'
# Output should return items formatted with [@CitationKeys].

# 3. Test Co-Researcher Skill
> /research
# Output should display the researcher prompt.
```
### 5. How to Use
Once connected, agents has access to specific tools like `zotero_semantic_search` and `zotero_get_item_fulltext`. You do not need to call these manually; you must invoke them in plain English but be SPECIFIC.

**Example Prompts:**
### Zotero (Literature & Citations)
*   **Semantic Discovery:** *"Find papers conceptually similar to 'deep learning in computer vision' even if they don't use those exact keywords."*
*   **Abstract Matching:** *"Find papers that discuss topics similar to this abstract: [paste abstract]"*
*   **Extraction:** *"Extract all PDF annotations from my paper on 'Social Networks' and summarize the key points."*
*   **Complex Filtering:** *"Show me papers tagged '#Boundaries' excluding those with '#Segregation'."*

### Notion (Notes & Summaries)
*   **Project Context:** *"Search Notion for my notes on 'London Housing Crisis' and summarize the current hypothesis."*
*   **Task Management:** *"Query my Notion database for all tasks tagged 'Literature Review' that are not yet marked 'Done'."*
*   **Concept Linking:** *"Read the page 'Theoretical Framework' in Notion and compare it with the findings from the Zotero paper I just found."*

### Combined Workflow (Co-Research)
*   **Synthesis:** *"Using the 'London Housing' notes from Notion and the 'Butler 2003' paper from Zotero, write a paragraph synthesizing how symbolic boundaries impact displacement."*
*   **Gap Analysis:** *"Review my 'Research Plan' in Notion. Based on the papers available in Zotero, what key literature on 'displacement' am I missing?"*

### Reseach plan
* /Research [TBC]

---
Corrections: If you see mistakes or want to suggest changes, please create an [issue] (https://github.com/Andrei-WongE/geminiCLI_hooks_and_extensions-4research/issues/new).

## License

MIT License - See [LICENSE](LICENSE) file for details.

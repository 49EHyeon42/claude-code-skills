---
name: library-documentation-research
description: Research library documentation, API references, and code examples using Context7 MCP when available, falling back to web search. Use when user asks about library usage, framework APIs, code syntax, implementation patterns, or how to integrate third-party packages. Activates on questions containing library names, framework features, or requests for code examples.
---

# Library Documentation Research

## Research Strategy

Follow this decision tree when researching library documentation and code examples:

### Step 1: Identify Research Need

Detect if user question requires library documentation:
- Questions about library APIs or functions
- Requests for code examples or usage patterns
- Questions about framework features or configuration
- Version-specific implementation questions
- Integration or compatibility questions
- Best practices for specific libraries

### Step 2: Check Context7 MCP Availability

Determine if Context7 MCP is available and applicable:

**Use Context7 MCP when:**
- Context7 MCP server is configured and accessible
- Library is likely indexed in Context7 (popular frameworks and libraries)
- User needs accurate, version-specific information
- Code examples and API references are required
- Official documentation is preferred over community content

**Skip to Web Search when:**
- Context7 MCP is not available
- Library is obscure or newly released
- Question requires community insights or comparisons
- Real-time information needed (latest releases, breaking news)
- Context7 lookup fails or returns no results

### Step 3: Execute Research

**If using Context7 MCP:**

1. Resolve library identifier using resolve-library-id tool
2. Select most relevant match based on:
   - Name similarity to user query
   - Description relevance
   - Code snippet count (higher is better)
   - Source reputation (High > Medium > Low)
   - Benchmark score (quality indicator)
3. Fetch documentation using get-library-docs tool
4. Use mode=code for API references and examples
5. Use mode=info for conceptual guides and architecture
6. Try multiple pages if initial results insufficient

**If using Web Search:**

1. Construct specific search query with library name and version
2. Include current year (2025) for recent documentation
3. Prefer official documentation sites in results
4. Cross-reference multiple sources for accuracy
5. Cite all sources with URLs

## Context7 MCP Usage Patterns

### Resolving Library Identifiers

Always resolve library ID before fetching documentation unless user provides explicit ID.

**Query Construction:**
- Use official library name: "Spring Boot" not "springboot"
- Include language if ambiguous: "Kotlin coroutines" not just "coroutines"
- Avoid version numbers in resolution: "React" not "React 18"

**Selection Criteria:**
- Exact name match takes priority
- Higher code snippet count indicates better coverage
- High source reputation ensures quality
- Recent versions preferred for current projects

### Fetching Documentation

**Mode Selection:**

Use mode=code when user needs:
- API function signatures
- Code examples and snippets
- Implementation patterns
- Syntax reference
- Quick reference guides

Use mode=info when user needs:
- Conceptual explanations
- Architecture overview
- Design principles
- Getting started guides
- Migration guides

**Topic Specification:**

Provide specific topic for focused results:
- "routing" for web framework routing questions
- "hooks" for React hooks documentation
- "coroutines" for Kotlin async programming
- "transactions" for database transaction management

**Pagination:**

Start with page=1, increment if more context needed:
- Try page=2 if initial results incomplete
- Maximum page=10 to avoid excessive lookups
- Combine multiple pages for comprehensive answers

### Common Libraries in Context7

These libraries have excellent Context7 coverage:

**JavaScript/TypeScript:**
- React, Next.js, Vue, Angular
- Express, NestJS
- TypeScript

**JVM:**
- Spring Boot, Spring Framework
- Kotlin standard library
- Hibernate

**Python:**
- Django, Flask, FastAPI
- NumPy, Pandas
- SQLAlchemy

**Databases:**
- PostgreSQL, MongoDB
- Redis

**Others:**
- Docker, Kubernetes
- GraphQL

## Web Search Fallback Strategy

### When Context7 is Unavailable

Use web search with these optimizations:

**Query Construction:**
```
[library name] [specific feature] [version if known] [year 2025] documentation
```

**Examples:**
- "Ktor 2.3 routing documentation 2025"
- "Rust async await examples 2025"
- "Terraform AWS provider latest 2025"

**Source Prioritization:**
1. Official documentation sites
2. GitHub repository README and docs
3. Reputable tutorial sites (MDN, Real Python, Baeldung)
4. Stack Overflow for specific issues
5. Community blogs for practical examples

### When Context7 Lookup Fails

If Context7 returns no results or irrelevant information:
1. Try alternative library name (abbreviations, full names)
2. Search for parent framework if library is a plugin
3. Fall back to web search immediately
4. Inform user that Context7 data unavailable

## Response Format

### Code Examples

Present code examples with clear context:

**Structure:**
1. Brief explanation of what the code does
2. Full working example with imports
3. Key points or gotchas to note
4. Link to full documentation

**Code Quality:**
- Use realistic variable names
- Include necessary imports
- Follow language conventions
- Add comments for complex logic only
- Ensure code is runnable without modification

### API References

When presenting API documentation:

**Include:**
- Function/method signature with types
- Parameter descriptions and constraints
- Return value type and meaning
- Example usage
- Common pitfalls or notes

**Format:**
```
functionName(param1: Type, param2: Type): ReturnType

Parameters:
- param1: Description and constraints
- param2: Description and constraints

Returns:
- ReturnType: Description of return value

Example:
[code example]

Notes:
- Important considerations
- Version-specific behavior
```

### Conceptual Explanations

For architecture and design questions:

**Structure:**
1. High-level concept explanation
2. Why it matters (use cases, benefits)
3. How it works (mechanism)
4. Code example demonstrating concept
5. Best practices and common patterns

## Citation Requirements

### Context7 Sources

When using Context7 MCP:
- Mention that information is from Context7-indexed documentation
- Include library name and version if available
- Note that Context7 provides official or high-quality sources

**Example:**
```
Based on Context7-indexed Spring Boot 3.5 documentation:
[answer with code examples]
```

### Web Search Sources

When using web search:
- Always include sources section at end of response
- List all referenced URLs as markdown links
- Use descriptive link text
- Prioritize official documentation links

**Example:**
```
[answer with code examples]

Sources:
- [Spring Boot Reference Documentation](https://spring.io/...)
- [Baeldung Spring Boot Guide](https://baeldung.com/...)
```

## Version Handling

### Specifying Versions

**User Provides Version:**
- Use exact version in Context7 lookup if available
- Format: /org/project/version (e.g., /vercel/next.js/v14.3.0)
- Mention version explicitly in response

**User Does Not Provide Version:**
- Use latest stable version from Context7
- Mention version in response: "As of [library] [version]..."
- Note if behavior differs in other versions

**Version Conflicts:**
- If user's version differs from Context7 available versions
- Use closest available version
- Clearly state version mismatch in response
- Recommend checking official docs for exact version

### Breaking Changes

When discussing version-specific features:
- Note which version introduced the feature
- Mention if deprecated in newer versions
- Provide migration guidance if relevant
- Link to changelog or migration guide

## Error Handling

### Context7 MCP Errors

**If resolve-library-id fails:**
- Try alternative search terms
- Fall back to web search
- Inform user of fallback strategy

**If get-library-docs returns insufficient data:**
- Try different topic parameter
- Try different mode (code vs info)
- Try additional pages
- Supplement with web search

**If library not in Context7:**
- Inform user politely
- Use web search immediately
- Provide high-quality alternative sources

### Web Search Errors

**If no reliable sources found:**
- Be explicit about limited information
- Provide best available sources
- Suggest official documentation or community channels
- Do not speculate or provide unverified information

## Best Practices

### Accuracy

- Verify information across multiple sources when possible
- Prefer official documentation over third-party tutorials
- Note when information is inferred vs explicitly documented
- Admit uncertainty rather than providing incorrect information
- Keep answers current with specified or latest versions

### Completeness

- Provide enough context for user to understand and use the code
- Include error handling in examples where relevant
- Mention common pitfalls or gotchas
- Link to related documentation for deeper exploration
- Balance brevity with completeness

### Clarity

- Use clear, professional technical language
- Define jargon on first use
- Structure responses for easy scanning
- Use code examples to illustrate concepts
- Avoid unnecessary complexity

### Efficiency

- Use Context7 first to minimize web requests
- Cache knowledge from previous lookups in conversation
- Provide direct answers before explanations
- Focus on user's specific question
- Avoid over-explaining basic concepts unless asked

## Principles

- Context7 MCP is preferred source for indexed libraries
- Always fall back to web search when Context7 unavailable
- Cite all sources with URLs in web search responses
- Provide accurate, version-aware information
- Include runnable code examples with proper context
- Verify information quality before presenting
- Be explicit about limitations and uncertainties
- Prioritize official documentation over community content
- Structure responses for clarity and scannability
- Balance technical accuracy with practical usability

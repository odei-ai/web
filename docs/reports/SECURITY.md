# ODEI Security Best Practices

## API Key Management

### ⚠️ CRITICAL: Never Commit API Keys

**Current Issue**: API keys are hardcoded in MCP configuration files:

- `.claude/mcp.json`
- `agents/*/. claude/mcp.json`

**These files contain OPENAI_API_KEY values that should NOT be committed to version control.**

### Recommended Approach

#### 1. Use Environment Variables (Future)

Once Claude Code supports env interpolation in `mcp.json`:

```json
{
  "mcpServers": {
    "odei-neo4j": {
      "env": {
        "OPENAI_API_KEY": "${OPENAI_API_KEY}"
      }
    }
  }
}
```

#### 2. Current Workaround

Until then:

**Option A**: `.gitignore` MCP configs

```bash
# .gitignore
.claude/mcp.json
agents/*/.claude/mcp.json
```

**Option B**: Template-based approach

```bash
# Commit .mcp.json.template files
# Each developer creates local .mcp.json from template
cp .claude/mcp.json.template .claude/mcp.json
# Fill in actual API keys locally
```

**Option C**: Encrypted secrets (Advanced)

```bash
# Use git-crypt or similar
git-crypt init
git-crypt add-gpg-user YOUR_GPG_KEY
echo ".claude/mcp.json filter=git-crypt diff=git-crypt" >> .gitattributes
```

### 3. .env File Pattern

Use `.env` for local development:

```bash
# Copy template
cp .env.example .env

# Edit with your keys
vim .env

# Ensure .env is in .gitignore
echo ".env" >> .gitignore
```

## Neo4j Security

### Database Credentials

Current credentials in configs:

```
NEO4J_USERNAME=neo4j
NEO4J_PASSWORD=juehxbr413390
```

**Recommendations**:

1. Change default password immediately
2. Use strong, unique passwords (20+ characters)
3. Consider environment variables for production
4. Enable Neo4j authentication

### Network Security

- Default: `bolt://127.0.0.1:7687` (localhost only)
- For remote access: Use TLS (`bolt+s://` or `neo4j+s://`)
- Firewall: Only allow trusted IPs

## OpenAI API Security

### API Key Best Practices

1. **Rotation**: Rotate keys every 90 days
2. **Least Privilege**: Create separate keys for dev/staging/prod
3. **Monitoring**: Enable usage alerts in OpenAI dashboard
4. **Rate Limits**: Set spending limits to prevent abuse

### Cost Control

Monitor embedding generation costs:

- text-embedding-3-large: $0.00013 per 1K tokens
- Average node: ~200 tokens = ~$0.000026 per embedding
- Set monthly budget alerts

## MCP Server Security

### Server Isolation

Each MCP server runs as separate process:

- `odei-neo4j`: Write operations (highest privilege)
- `*-neo4j` (agents): Read-only analytics (lower privilege)

**Principle**: Minimize write permissions, maximize read isolation

### Tool Allow Lists

Always use `toolAllowList` in MCP configs:

```json
{
  "toolAllowList": ["odei.neo4j.value.create.v1", "odei.neo4j.value.update.v1"]
}
```

Prevents accidental exposure of dangerous operations.

## Audit Trail

All operations include provenance:

```json
{
  "provenance": {
    "module": "discuss",
    "actor": "agent",
    "source": "constitutional-validation",
    "timestamp": "2025-10-24T15:16:01.388Z"
  }
}
```

**Benefits**:

- Track who changed what
- Detect unauthorized modifications
- Debug data quality issues

## Secrets Checklist

Before committing code:

- [ ] No API keys in source code
- [ ] No passwords in config files
- [ ] `.env` file in `.gitignore`
- [ ] `.env.example` template provided
- [ ] MCP configs use placeholders or excluded from git
- [ ] README warns about secret management
- [ ] Pre-commit hook checks for secrets (optional)

## Incident Response

If API key is compromised:

1. **Immediate**: Revoke key in OpenAI dashboard
2. **Generate new key** and update local configs
3. **Rotate Neo4j password** if exposed
4. **Audit**: Check usage logs for unauthorized access
5. **Update** `.gitignore` to prevent future leaks

## Security Contacts

- OpenAI Security: https://openai.com/security
- Neo4j Security: https://neo4j.com/security
- Report ODEI security issues: [Add your contact]

---

**Last Updated**: 2025-10-24
**Review Frequency**: Quarterly

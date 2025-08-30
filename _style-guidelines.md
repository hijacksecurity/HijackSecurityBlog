# Hijack Security Blog - Style Guidelines

## Blog Design Philosophy

This blog maintains a **professional, technical aesthetic** focused on security, development, and infrastructure topics. The design prioritizes readability, scanability, and professional appearance over visual decoration.

## Formatting Standards

### Emoji Usage
- **Minimal and purposeful only** - use sparingly for section headers when it adds genuine value
- **Maximum 1-2 emojis per heading** when used
- **Avoid decorative emojis** in body text and bullet points
- **Professional context** - only use when appropriate for technical content

### Heading Structure
- Use standard markdown hierarchy: `##` for main sections, `###` for subsections
- Keep headings clean and descriptive
- Avoid excessive visual decoration in headings

### Text Formatting
- Use **bold** for emphasis on key concepts and terms
- Use *italics* for quotes, notes, and subtle emphasis
- Use `code formatting` for technical terms, commands, and file names
- Use standard markdown formatting over HTML when possible

### Lists and Bullets
- Use clean bullet points with **bold** keywords followed by descriptions
- Keep bullet points scannable and concise
- Avoid emojis in bullet points unless absolutely necessary

### Code Blocks
- Use proper language specification for syntax highlighting
- Include clear, practical examples
- Add comments when helpful for understanding

### Links
- Use descriptive link text
- Format as `[descriptive text](url)` in standard markdown
- Avoid "click here" or generic link text

## Content Structure

### Post Layout
1. **Clear introduction** explaining what the post covers
2. **Logical section flow** with descriptive headings
3. **Practical examples** and code when relevant
4. **Clear conclusion** with next steps or key takeaways

### Technical Content
- **Security-focused perspective** throughout
- **Production-ready examples** over toy demonstrations
- **Real-world context** and practical applications
- **Clear explanations** of technical concepts

## Visual Elements

### Images
- Use images purposefully to illustrate concepts
- Include descriptive alt text
- Use consistent styling when adding custom HTML

### HTML Usage
- Keep custom HTML minimal
- Use only when markdown is insufficient
- Maintain consistent styling patterns
- Focus on functionality over decoration

## Consistency Examples

### Good Examples (Follow These Patterns)
```markdown
## Setting Up Infrastructure

The first step involves configuring your EKS cluster with security controls built in from the foundation.

### Key Configuration Steps

- **Cluster Authentication** - Configure IAM roles for secure access
- **Network Security** - Implement security groups and network policies
- **Monitoring Setup** - Enable comprehensive logging and metrics

```bash
# Configure the cluster
eksctl create cluster --config-file cluster.yaml
```

### Poor Examples (Avoid These Patterns)
```markdown
## 🚀✨ Setting Up Infrastructure 🔥💪

The first step involves configuring your EKS cluster with security controls built in from the foundation! 🎯

### Key Configuration Steps 🛠️

- 🔐 **Cluster Authentication** - Configure IAM roles for secure access! 
- 🛡️ **Network Security** - Implement security groups and network policies!
- 📊 **Monitoring Setup** - Enable comprehensive logging and metrics!

```bash
# Configure the cluster 🚀
eksctl create cluster --config-file cluster.yaml
```

## Quality Checklist

Before publishing, ensure:

- [ ] Minimal emoji usage (0-2 per heading max)
- [ ] Professional tone throughout
- [ ] Clear heading hierarchy
- [ ] Scannable bullet points
- [ ] Proper code formatting
- [ ] Descriptive link text
- [ ] Consistent with existing posts
- [ ] Technical accuracy and security focus

## Reference Posts

Use these existing posts as style references:
- `2025-07-12-welcome-to-hijack-security-blog.md` - Clean, minimal style
- `2025-08-25-building-aicouncil-multi-agent-conversation-system.md` - Technical depth with appropriate HTML usage

## Final Notes

The goal is to maintain a professional technical blog that developers and security professionals want to read and reference. Clean, scannable formatting helps readers focus on the valuable technical content rather than getting distracted by visual clutter.
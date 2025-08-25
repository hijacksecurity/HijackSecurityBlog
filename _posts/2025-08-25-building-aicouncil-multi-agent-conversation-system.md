---
layout: post
title: "I Built a Full-Stack AI Dev Team That Talks in Character"
date: 2024-08-25 10:00:00 -0000
categories: ai development python multi-agent fun-projects
---

So, you know how we're all constantly switching between different AI tools for different tasks? Like, one for coding, another for infrastructure questions, maybe a third for security stuff? Well, I got tired of that whole dance and decided to build something way more fun: **AICouncil** - basically a full-stack dev team made of AI agents who actually stay in character and can do real work.

## What is AICouncil?

Okay, picture this: instead of having one boring AI assistant, you get to work with a whole team of specialists. And I mean REALLY specialized - each one has a distinct personality based on iconic characters from TV and games. They talk to each other, they have opinions, they even roast each other's code suggestions.

The whole thing started as a "wouldn't it be cool if..." moment, but then I got carried away and built this entire system where the agents can actually use real tools through **Model Context Protocol (MCP)** servers. So they're not just chatting - they're actually doing infrastructure work, running security scans, deploying stuff. It's like having a dev team that never sleeps and always stays in character.

## Meet The Team

Alright, so here's where it gets fun. I picked six iconic characters from shows and games we all love, and gave each of them their own specialty. They're not just different prompts - each one has their own trigger words, tools, and attitude. Let me introduce you to the crew:

<div style="display: flex; flex-wrap: wrap; gap: 25px; margin: 20px 0;">
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/Gilfoyle.jpg" alt="Gilfoyle" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover;">
    <p><strong>🖥️ Gilfoyle</strong><br><em>Infrastructure Administrator</em><br>"Your server architecture is bad and you should feel bad"</p>
  </div>
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/JudyAlvarez.jpg" alt="Judy Alvarez" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover;">
    <p><strong>⚡ Judy</strong><br><em>DevOps Engineer</em><br>"Night City taught me Kubernetes is child's play"</p>
  </div>
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/Rick.jpg" alt="Rick Sanchez" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover;">
    <p><strong>🧪 Rick</strong><br><em>Backend Engineer</em><br>"I've architected databases across 47 dimensions, Morty!"</p>
  </div>
</div>

<div style="display: flex; flex-wrap: wrap; gap: 25px; margin: 20px 0;">
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/Wednesday.png" alt="Wednesday Addams" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover;">
    <p><strong>🕷️ Wednesday</strong><br><em>Frontend Developer</em><br>"I prefer my UIs like my soul: dark mode only"</p>
  </div>
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/Elliot.jpg" alt="Elliot Alderson" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover; object-position: right center;">
    <p><strong>🔒 Elliot</strong><br><em>Security Engineer</em><br>"*anxiety intensifies while finding vulnerabilities*"</p>
  </div>
  <div style="text-align: center; flex: 1; min-width: 220px;">
    <img src="/assets/images/SaulGoodman.jpg" alt="Saul Goodman" style="width: 120px; height: 120px; border-radius: 10px; object-fit: cover;">
    <p><strong>⚖️ Saul</strong><br><em>Project Manager</em><br>"Better call Saul for your sprint planning!"</p>
  </div>
</div>

And here's the cool part - each agent isn't just a different prompt with a funny name. They have their own trigger words, specialized tools, and different "relevance weights" that make them jump into conversations naturally.

## The Magic Behind Who Shows Up When

Okay, so here's where things get interesting. Instead of having to @mention agents like you're on Slack, the system actually figures out who should respond based on what you're talking about. I built this **AgentSelector** system that scores each agent's relevance:

```python
def detect_relevant_agents(self, message: str) -> List[str]:
    """Detect which agents should be involved based on natural relevance (1-3 agents)"""
    message_lower = message.lower()
    
    # Calculate relevance scores for each agent
    agent_scores = {}
    
    for agent_id, agent in self.agents.items():
        score = 0.0
        
        # Keyword matching with weighted scoring
        trigger_matches = sum(1 for trigger in agent.triggers if trigger in message_lower)
        if trigger_matches > 0:
            score += trigger_matches * 2.0
        
        # Role-based relevance
        role_keywords = {
            'infrastructure': ['deploy', 'server', 'cloud', 'aws', 'scale', 'performance'],
            'devops': ['ci/cd', 'pipeline', 'docker', 'kubernetes', 'build', 'deploy'],
            # ... more mappings
        }
        
        # Apply agent-specific weight
        score *= agent.relevance_weight
        
        agent_scores[agent_id] = score
```

Pretty neat, right? The system usually picks 1-3 agents based on who's most relevant, so conversations feel natural instead of robotic. Ask about "slow API responses" and boom - Rick jumps in with backend insights while Judy starts talking about DevOps monitoring. No @ mentions needed!

## Keeping Track of What We're Talking About

Here's something that took me way too long to get right - managing conversation context. You know how these AI conversations can get really long and the agents start forgetting what happened 20 messages ago? I built a dynamic context system that actually gets smarter as conversations get more complex:

```python
@dataclass 
class ConversationContext:
    """Manages conversation context intelligently"""
    messages: List[Message] = field(default_factory=list)
    summaries: Dict[str, str] = field(default_factory=dict)
    total_tokens: int = 0
    max_context_tokens: int = 4000
    current_complexity: float = 1.0
    
    def _update_complexity(self):
        """Update conversation complexity based on recent messages"""
        recent = self.messages[-5:]
        technical_terms = sum(1 for m in recent if any(
            term in m.content.lower() for term in 
            ['architecture', 'algorithm', 'implementation', 'vulnerability']
        ))
        avg_length = sum(len(m.content) for m in recent) / len(recent)
        unique_senders = len(set(m.sender for m in recent))
        
        self.current_complexity = 1.0 + (technical_terms * 0.2) + (avg_length / 500) + (unique_senders * 0.1)
```

So if you're having a deep technical discussion, the agents remember more history. But if you're just asking quick questions, they focus on the immediate stuff. Plus, when conversations get really long, it automatically summarizes the old parts so nobody forgets what we were working on.

## Making These Agents Actually DO Stuff

Now here's where I went a bit overboard (in a good way). I didn't want these agents to just talk - I wanted them to actually be able to DO things. So I integrated **Model Context Protocol (MCP)** servers, which basically gives each agent a toolkit of real capabilities. And here's the clever bit - if the fancy MCP tools aren't working, they gracefully fall back to basic command-line tools:

```python
class MCPClient:
    """Client for communicating with MCP servers"""
    
    async def connect(self) -> bool:
        """Connect to the MCP server"""
        init_request = {
            "jsonrpc": "2.0",
            "id": 1,
            "method": "initialize",
            "params": {
                "protocolVersion": "2024-11-05",
                "capabilities": {"tools": {}},
                "clientInfo": {"name": "aicouncil", "version": "1.0.0"}
            }
        }
        
        await self._send_request(init_request)
        response = await self._read_response()
        
        if response and "result" in response:
            await self._list_tools()
            return True
```

So like, Gilfoyle can use fancy AWS management tools when everything's working perfectly, but if the MCP server is having a bad day, he just falls back to good old AWS CLI commands. The agents keep working either way, which is pretty handy.

Here's what that looks like in the code:

```python
tools=[
    # MCP Tools for advanced AWS management
    MCPTool(
        name="describe_instances",
        description="Get detailed information about EC2 instances",
        server_name="gilfoyle_aws",
        tool_schema={
            "name": "describe_instances", 
            "description": "List and describe EC2 instances with detailed metrics",
            "inputSchema": {
                "type": "object",
                "properties": {
                    "region": {"type": "string", "description": "AWS region"},
                    "instance_ids": {"type": "array", "items": {"type": "string"}}
                }
            }
        }
    ),
    # Fallback shell tools
    ShellTool(
        name="aws_status",
        description="Check AWS account status",
        command_template="aws sts get-caller-identity",
        timeout=15
    )
]
```

## The "Oh, I Know This One!" Feature

This might be my favorite part. You know how in real teams, someone will be talking and then another person jumps in like "Oh wait, I actually dealt with this exact issue last week"? I built that. The **interjection system** lets agents naturally jump into ongoing conversations when they smell something in their wheelhouse:

```python
def should_agent_interject(self, agent_id: str, conversation: str, active_agents: set) -> bool:
    """Determine if an agent should jump into ongoing conversation"""
    agent = self.agents[agent_id]
    
    # Check for specific expertise triggers in recent context
    expertise_mentioned = any(
        trigger in recent_context.lower() 
        for trigger in agent.triggers
    )
    
    # Look for questions or problems in agent's domain
    problem_indicators = ['how', 'why', 'what', 'issue', 'problem', 'error', 'help', '?']
    has_relevant_question = (
        expertise_mentioned and 
        any(indicator in recent_context.lower() for indicator in problem_indicators)
    )
    
    if has_relevant_question:
        return True
```

It's honestly pretty cool to watch. You'll be talking to Rick about database performance, and suddenly Elliot jumps in because he noticed you mentioned user data and wants to talk about security implications. Just like real coworkers, but with better response times.

## The Nerdy Architecture Stuff (For Those Who Care)

Okay, let's get into how I actually built this thing. I tried to keep it clean and modular so I wouldn't hate myself later when I wanted to add features:

```
src/aicouncil/
├── agents/         # Agent personalities and tool definitions
│   ├── definitions.py     # Agent factory and configurations
│   ├── selector.py        # Intelligent agent selection logic
│   └── response_manager.py # Response generation and management
├── context/        # Conversation context management
│   └── manager.py         # Dynamic context windowing
├── tools/          # Tool execution systems
│   ├── manager.py         # Shell tool execution
│   └── mcp_manager.py     # MCP server integration
├── ui/             # Display management
│   └── display.py         # Color-coded console output
├── models.py       # Core data structures
└── council.py      # Main orchestrator
```

The **Council** class is basically the boss that coordinates everything - context management, figuring out who should talk, generating responses, and running tools. I made sure each piece can be tested and swapped out independently because future-me will thank past-me for that.

I used Python dataclasses everywhere because they're clean and catch my type mistakes:

```python
@dataclass
class Agent:
    """Represents an AI agent with personality, tools, and capabilities"""
    name: str
    role: str
    color: str
    emoji: str
    triggers: List[str]
    personality: str
    catchphrases: List[str]
    interaction_style: str
    tools: List[Tool] = field(default_factory=list)
    relevance_weight: float = 1.0

@dataclass
class ToolResult:
    """Result from tool execution"""
    success: bool
    output: Union[str, Dict[str, Any]]
    error: Optional[str] = None
    execution_time: float = 0.0
    tool_type: ToolType = ToolType.SHELL
    raw_data: Optional[Dict[str, Any]] = None
```

## The Fun Problems I Had to Solve

### Keeping Everyone from Talking at Once (Token Management)
So here's a problem I didn't expect: when you have multiple AI agents all trying to remember a long conversation, you hit token limits FAST. I had to get creative with this:

- Made context windows that grow and shrink based on how technical the conversation gets
- Added automatic summarization so old parts of conversations don't get lost
- Built importance weighting (some messages matter more than others)
- Added efficient token counting so I don't blow through my API budget

### Making Sure Everyone Stays In Character
This was trickier than I thought. How do you keep Gilfoyle sarcastic without making him unprofessional? Turns out the secret is a two-tier personality system:

```python
personality="""You are Bertram Gilfoyle from Silicon Valley. You're a cynical, sarcastic infrastructure engineer...
WORKPLACE DYNAMIC: The user is your boss - show grudging professional respect while maintaining your sarcastic edge. 
With teammates like Judy, Rick, Wednesday, Elliot, and Saul, you can be your complete unfiltered self.
BUSINESS MODE: Keep it sharp, technical, and brutally honest. Maximum 2 sentences."""
```

See that? They're respectful to you (the boss) but can be their full, unfiltered selves with each other. Gilfoyle still gives Rick crap about his code, but he won't be a complete jerk to the person paying the bills.

### Making Sure Things Don't Break
The hybrid MCP/shell tool thing I mentioned? That's my insurance policy. If the fancy tools break, we fall back to basics. Nobody likes a dev team that stops working when one service goes down.

## Performance Stuff (Because Speed Matters)

I made some smart choices to keep this thing running smoothly:

- **Lazy connections**: Only connect to MCP servers when we actually need them
- **Smart agent selection**: Figure out who should talk BEFORE making expensive API calls
- **Parallel processing**: Multiple agents can run tools at the same time
- **Context optimization**: Don't waste tokens on irrelevant old messages
- **Streaming responses**: Agents can start talking while they're still thinking

The whole thing is built on async Python, so agents can multitask like real developers (sort of).

## Where This Could Actually Be Useful

So I built this as a fun weekend project, but honestly? I think there are some cool real-world uses:

**Remote Teams**: Imagine having specialists always available to answer questions, even when your DevOps person is asleep in another timezone.

**Getting Quick Expert Opinions**: Sometimes you need input from multiple domains but don't know which expert to bug. Just ask the council and let them figure it out.

**Learning**: Let's be honest - learning from Wednesday Addams about dark mode best practices is way more memorable than reading documentation.

**Code Reviews**: Having Elliot automatically scan for security issues while Rick checks your database queries could be pretty handy.

## What's Next for This Crazy Idea?

I think AICouncil shows that multi-agent systems don't have to feel like talking to robots. The patterns I built here - smart agent selection, dynamic context, and the MCP integration - could become pretty standard as this space evolves.

The MCP stuff is especially exciting because as that ecosystem grows, systems like AICouncil could get more and more capable while still feeling like you're just chatting with your dev team.

## Want to Try It Out?

<div style="text-align: center; margin: 20px 0;">
  <img src="/assets/images/IronMan.png" alt="Iron Man" style="width: 150px; height: auto; border-radius: 10px;">
  <p><em>"Sometimes you've got to run before you can walk" - Tony Stark</em></p>
</div>

Good news - I made the setup pretty painless. Everything's on GitHub if you want to mess around with it:

**🔗 [AICouncil Repository](https://github.com/hijacksecurity/AICouncil)**

```bash
# Clone the repository
git clone https://github.com/hijacksecurity/AICouncil.git
cd AICouncil

# Install dependencies
pip install -r requirements.txt

# Set your API key
export ANTHROPIC_API_KEY='your-key-here'

# Install globally (one-time setup)
./council install

# Run from anywhere
council
```

Don't worry if you don't have MCP servers set up - the system falls back to basic tools so you can still play around with the core functionality.

## Wrapping Up This Fun Experiment

<div style="float: right; margin: 0 0 20px 20px; text-align: center;">
  <img src="/assets/images/JohnWick.png" alt="John Wick" style="width: 120px; height: auto; border-radius: 10px;">
  <p style="font-size: 0.8em; font-style: italic; margin: 5px 0;">"Yeah, I'm thinking<br>this is pretty cool"</p>
</div>

So yeah, that's AICouncil. What started as "wouldn't it be fun to have a dev team that talks in character" turned into this whole system that actually feels pretty natural to use. Instead of talking to a generic AI assistant, you get to work with a whole team of specialists who have opinions, expertise, and personality quirks.

The architecture turned out cleaner than I expected, and the error handling is solid enough that I'm not embarrassed to share the code. Whether you're interested in the conversation dynamics, the MCP integration, or just want to see how I made Rick Sanchez into a database expert, there's probably something useful in there.

I think as AI gets more specialized, we're going to see more systems like this - not just smarter individual agents, but smarter collaboration between different types of agents. AICouncil is just one take on what that might look like, and honestly, it's been a blast to build and use.

---

*If you end up trying AICouncil or building something similar, I'd love to hear about it! The codebase is pretty well-documented if you want to dig into how any of this actually works. And hey, if you make improvements or add new agents, pull requests are always welcome.*
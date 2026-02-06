---
layout: post
title: Vibe Programming (1) - Qwen Code
toc: true
categories: [AI, Programming]
---

As a developer, I've been using various AI tools in my work to gather knowledge, troubleshoot issues, and polish code. My usual suspects have been ChatGPT and DeepSeek, which have served me well for general queries and code suggestions. However, when I recently worked on a project that was almost entirely written with the help of ChatGPT and GitHub Copilot, I hit a wall – exceeding my quota limits. This led me to search for a viable alternative that could meet my specific requirements.

<!--more-->

## Background: My Experience with AI Programming Tools

For quite some time, ChatGPT and GitHub Copilot have been my go-to tools for:
- Gathering technical knowledge and best practices
- Troubleshooting complex issues
- Polishing existing code
- Getting suggestions for improvements

However, when I exceeded my quota limits on these platforms, I realized I needed an alternative solution that could offer similar capabilities without the recurring costs or limitations.

## Requirements for My Ideal AI Programming Tool

After evaluating my workflow and needs, I identified three key requirements for an AI programming tool:

1. **Free Access**: Since I don't use AI programming tools daily, I didn't want to commit to a paid subscription plan.
2. **Direct File Manipulation**: The tool should be able to create and modify local files directly, not just provide advice or patch files.
3. **High Quality Results**: The tool should provide accurate, reliable, and contextually appropriate code suggestions.

## Exploring Alternatives

I considered several options in the market:
- Amazon CodeWhisperer
- Claude Code
- Windsurf (formly Codeium)
- And others

While each had its strengths, none perfectly matched all my requirements until I discovered Qwen Code.

## Why I Chose Qwen Code

Qwen Code stood out because it met all my requirements:

### Free Access with Generous Quota
Qwen Code offers a generous free tier that allows developers to experiment and use the tool extensively without worrying about subscription costs. This is perfect for occasional users who don't need daily access to AI programming assistance.

### Direct File Creation and Modification
Unlike many other AI tools that only provide code snippets or patches, Qwen Code enables direct creation and modification of local files. This capability significantly streamlines the development workflow, allowing for seamless integration into the coding process.

### High-Quality Results
The quality of code suggestions and completions from Qwen Code has been impressive, often matching or exceeding what I experienced with other premium tools.

## What is Qwen Code?

Qwen Code is an AI-powered coding assistant developed by Alibaba Cloud. It's designed to help developers write code more efficiently by providing intelligent suggestions, auto-completions, and code generation capabilities. The tool understands context, follows best practices, and can work with multiple programming languages.

### Key Features of Qwen Code:
- **Intelligent Code Completion**: Understands context and provides relevant suggestions
- **Multi-language Support**: Works with various programming languages
- **Natural Language Understanding**: Can interpret plain English requests for code generation
- **Local File Operations**: Can create, modify, and manage local files directly
- **Integration Capabilities**: Seamlessly integrates with popular development environments

### Free Quota Information
Qwen Code offers a substantial free tier that includes:
- A generous number of tokens/month for code generation and completion
- Access to core features without payment requirements
- Sufficient capacity for occasional to moderate usage

Currently, it is [free, with a quota of 60 requests/minute and 2,000 requests/day](https://qwenlm.github.io/qwen-code-docs/en/users/configuration/auth/).

![Qwen Code Quota]({{ "/images/2026-01-18-qwen-code-quota.png" | absolute_url }}){:width="100%"}{:.glightbox}

## How to Use Qwen Code with VS Code: Step-by-Step Guide

Here's how to get started with Qwen Code in Visual Studio Code:

### Step 1: Install the Qwen Code Extension
1. Open Visual Studio Code
2. Navigate to the Extensions marketplace (Ctrl+Shift+X)
3. Search for "Qwen Code Companion"
4. Click "Install" to add the extension to your VS Code

![Qwen Code Companion]({{ "/images/2026-01-18-vscode-qwen-code.png" | absolute_url }}){:width="100%"}{:.glightbox}

### Step 2: Configure Your Account
1. Once installed, open the Qwen Code panel (usually accessible from the sidebar)
2. Sign up for an account or log in if you already have one
3. Follow the authentication process to link your VS Code with your Qwen Code account

### Step 3: Start Using Qwen Code
1. Open any code file in your editor
2. Open the Qwen Code Chat panel using one of these methods:
   - Click the Qwen icon in the top-right corner of the editor. ![]({{ "/images/2026-01-18-qwen-code-icon.png" | absolute_url }}){:width="10%"}{:.glightbox}
   - Run "Qwen Code: Open" from the Command Palette (Cmd+Shift+P / Ctrl+Shift+P)
![Qwen Code: Open]({{ "/images/2026-01-18-qwen-code-open.png" | absolute_url }}){:width="100%"}{:.glightbox}
3. Start chatting with Qwen - ask it to help with coding tasks, explain code, fix bugs, or write new features

![Qwen Code: chat dialog]({{ "/images/2026-01-18-qwen-code-chat-dialog.png" | absolute_url }}){:width="100%"}{:.glightbox}

### Step 4: Advanced Features
- Use the chat interface to ask complex coding questions and get detailed responses
- Select code and ask Qwen to explain, improve, or debug it
- Request Qwen to write tests for your selected code
- Generate documentation for functions and classes using natural language descriptions
- Leverage the context awareness to work with your entire project structure
- Use the inline suggestions when typing for real-time code completion

![Qwen Code: output]({{ "/images/2026-01-18-qwen-code-blog.png" | absolute_url }}){:width="100%"}{:.glightbox}

## Conclusion

Qwen Code has become my preferred AI programming assistant due to its combination of free access, direct file manipulation capabilities, and high-quality results. It fills the gap left by other tools when quotas are exceeded, and its generous free tier makes it ideal for developers who need occasional access to powerful AI coding assistance.

The tool's integration with VS Code and its ability to understand context and requirements makes it a valuable addition to any developer's toolkit. Whether you're looking to enhance productivity, learn new concepts, or simply need help with complex coding challenges, Qwen Code proves to be a capable and cost-effective solution.

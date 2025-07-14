# ai_project
The goal is to create a Python script that takes a Reddit user's profile URL, scrapes their posts and comments, and then constructs a user persona based on this data, citing the source content.
# Reddit User Persona Generator

This script scrapes a Reddit user's comments and posts to generate a detailed user persona, citing the source content used for each deduction.

## Features

* Scrapes Reddit comments and posts using PRAW.
* Utilizes a Large Language Model (LLM) (e.g., OpenAI's GPT) to synthesize a user persona.
* Cites specific Reddit content URLs for each persona characteristic.
* Outputs the persona to a text file.

## Technologies Used

* Python 3.8+
* PRAW (Python Reddit API Wrapper)
* OpenAI Python Client (for LLM interaction)
* Standard Python libraries (`os`, `re`, `sys`)

## Setup Instructions

### 1. Clone the Repository

```bash
git clone [https://github.com/yourusername/reddit-persona-generator.git](https://github.com/yourusername/reddit-persona-generator.git)
cd reddit-persona-generator

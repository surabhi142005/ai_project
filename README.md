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


# reddit_persona_generator.py

import praw
import os
import re
# Potentially import pandas for data handling if needed
# from transformers import pipeline # If using local NLP models
from openai import OpenAI # Or from google.generativeai import GenerativeModel

# --- Configuration ---
# You'll need to set up a Reddit API application to get these credentials.
# More info: https://praw.readthedocs.io/en/stable/getting_started/quick_start.html
REDDIT_CLIENT_ID = os.getenv("REDDIT_CLIENT_ID")
REDDIT_CLIENT_SECRET = os.getenv("REDDIT_CLIENT_SECRET")
REDDIT_USER_AGENT = "RedditPersonaGenerator/1.0 (by /u/YOUR_REDDIT_USERNAME)" # Replace with your Reddit username

# OpenAI API Key
OPENAI_API_KEY = os.getenv("OPENAI_API_KEY") # Or GOOGLE_API_KEY for Gemini

# Initialize Reddit API
try:
    reddit = praw.Reddit(
        client_id=REDDIT_CLIENT_ID,
        client_secret=REDDIT_CLIENT_SECRET,
        user_agent=REDDIT_USER_AGENT
    )
except Exception as e:
    print(f"Error initializing PRAW: {e}")
    print("Please ensure REDDIT_CLIENT_ID, REDDIT_CLIENT_SECRET, and REDDIT_USER_AGENT are set correctly.")
    exit()

# Initialize OpenAI Client
try:
    openai_client = OpenAI(api_key=OPENAI_API_KEY) # Or initialize Gemini: model = GenerativeModel('gemini-pro')
except Exception as e:
    print(f"Error initializing OpenAI client: {e}")
    print("Please ensure OPENAI_API_KEY is set.")
    exit()

# --- Functions ---

def get_redditor_username_from_url(url):
    """Extracts the username from a Reddit profile URL."""
    match = re.search(r"reddit\.com/user/([^/]+)", url)
    if match:
        return match.group(1)
    return None

def scrape_redditor_content(username, limit=100):
    """
    Scrapes comments and posts for a given Reddit username.
    Returns a dictionary with 'comments' and 'posts' lists.
    """
    redditor_content = {"comments": [], "posts": []}
    try:
        redditor = reddit.redditor(username)

        # Scrape comments
        for comment in redditor.comments.new(limit=limit):
            redditor_content["comments"].append({
                "id": comment.id,
                "url": f"https://www.reddit.com{comment.permalink}",
                "text": comment.body,
                "subreddit": comment.subreddit.display_name,
                "created_utc": comment.created_utc
            })

        # Scrape posts (submissions)
        for submission in redditor.submissions.new(limit=limit):
            redditor_content["posts"].append({
                "id": submission.id,
                "url": submission.url, # URL of the post itself
                "title": submission.title,
                "text": submission.selftext, # Body of self-text posts
                "subreddit": submission.subreddit.display_name,
                "created_utc": submission.created_utc
            })
        print(f"Scraped {len(redditor_content['comments'])} comments and {len(redditor_content['posts'])} posts for u/{username}")
    except Exception as e:
        print(f"Error scraping content for u/{username}: {e}")
    return redditor_content

def generate_user_persona(scraped_data):
    """
    Generates a user persona using an LLM based on scraped Reddit content.
    Includes citations to original posts/comments.
    """
    all_text = []
    citations = {}

    # Combine all relevant text and create a mapping for citations
    for content_type in ["comments", "posts"]:
        for item in scraped_data[content_type]:
            text = item.get("text") or item.get("title") # Use title for posts if no selftext
            if text:
                all_text.append(f"<{item['id']}>: {text}")
                citations[item['id']] = item['url']

    if not all_text:
        return "No sufficient text found to generate a persona."

    combined_input = "\n".join(all_text)

    # LLM Prompt Engineering
    # This prompt is crucial. You'll need to refine it for best results.
    # Instruct the LLM to include specific persona characteristics and to cite.
    prompt = f"""
    Analyze the following Reddit user's comments and posts to create a detailed user persona.
    The persona should include:
    - **Demographics:** (e.g., inferred age range, gender if clear, location if mentioned)
    - **Interests & Hobbies:** (What topics do they frequently discuss? What are their passions?)
    - **Values & Beliefs:** (What's important to them? What opinions do they express?)
    - **Personality Traits:** (Are they helpful, sarcastic, analytical, humorous?)
    - **Goals & Challenges:** (What are they trying to achieve or what problems do they face?)
    - **Online Behavior:** (How do they interact on Reddit? What subreddits do they frequent?)

    For each piece of information in the persona, you MUST cite the specific comment or post ID (e.g., "<comment_id>") that supports your deduction. If you infer something from multiple sources, cite all relevant IDs.

    Here is the Reddit content, with each item prefixed by its unique ID:

    {combined_input}

    ---
    Please generate the user persona below:
    """

    try:
        # For OpenAI
        response = openai_client.chat.completions.create(
            model="gpt-4o", # Or "gpt-3.5-turbo", "gemini-pro" for Google
            messages=[
                {"role": "system", "content": "You are a helpful assistant that analyzes user text to create detailed personas."},
                {"role": "user", "content": prompt}
            ],
            temperature=0.7,
            max_tokens=2000
        )
        persona_text = response.choices[0].message.content

        # Post-processing to replace IDs with full URLs for citations
        final_persona = persona_text
        for c_id, c_url in citations.items():
            final_persona = final_persona.replace(f"<{c_id}>", f"({c_url})")
        return final_persona

    except Exception as e:
        print(f"Error generating persona with LLM: {e}")
        return "Could not generate persona due to an error."

def save_persona_to_file(username, persona_data):
    """Saves the generated user persona to a text file."""
    filename = f"{username}_persona.txt"
    try:
        with open(filename, "w", encoding="utf-8") as f:
            f.write(persona_data)
        print(f"User persona saved to {filename}")
    except Exception as e:
        print(f"Error saving persona to file: {e}")

# --- Main Execution ---
if __name__ == "__main__":
    import sys

    if len(sys.argv) < 2:
        print("Usage: python reddit_persona_generator.py <reddit_profile_url>")
        print("Example: python reddit_persona_generator.py https://www.reddit.com/user/kojied/")
        sys.exit(1)

    profile_url = sys.argv[1]
    username = get_redditor_username_from_url(profile_url)

    if not username:
        print(f"Invalid Reddit profile URL: {profile_url}")
        sys.exit(1)

    print(f"Processing Reddit user: u/{username}")

    scraped_content = scrape_redditor_content(username)

    if scraped_content["comments"] or scraped_content["posts"]:
        user_persona = generate_user_persona(scraped_content)
        print("\n--- Generated User Persona ---")
        print(user_persona)
        save_persona_to_file(username, user_persona)
    else:
        print(f"No content found for u/{username}. Cannot generate persona.")

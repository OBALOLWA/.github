"""
spaces_bot.py
Simple Twitter/X Spaces watcher that tweets when a matching Space goes live.

Notes:
- Requires Tweepy v4+ (supports X API v2).
- Set environment variables before running:
    X_BEARER_TOKEN, X_API_KEY, X_API_SECRET, X_ACCESS_TOKEN, X_ACCESS_SECRET
"""

import os
import time
import logging
from typing import List, Set
import tweepy
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# CONFIG: keywords to watch for in space titles (case-insensitive)
WATCH_KEYWORDS = ["web3", "blockchain", "sui", "productivity"]

# Poll interval (hours)
POLL_INTERVAL = 2

# Environment credentials
BEARER_TOKEN = os.getenv("X_BEARER_TOKEN")  # read only -> spaces lookup/search
API_KEY = os.getenv("X_API_KEY")            # required for posting tweets
API_SECRET = os.getenv("X_API_SECRET")
ACCESS_TOKEN = os.getenv("X_ACCESS_TOKEN")
ACCESS_SECRET = os.getenv("X_ACCESS_SECRET")

if not BEARER_TOKEN:
    raise RuntimeError("Set X_BEARER_TOKEN in environment")

# Init Tweepy client for reads (bearer) and for write (user auth)
client_read = tweepy.Client(bearer_token=BEARER_TOKEN, wait_on_rate_limit=True)

# For posting tweets we use OAuth1 user auth -> tweepy.Client with user_tokens
client_write = None
if API_KEY and API_SECRET and ACCESS_TOKEN and ACCESS_SECRET:
    client_write = tweepy.Client(
        consumer_key=API_KEY,
        consumer_secret=API_SECRET,
        access_token=ACCESS_TOKEN,
        access_token_secret=ACCESS_SECRET,
        wait_on_rate_limit=True,
    )
else:
    logger.warning("Write credentials not found; bot will only detect but not tweet.")

def search_live_spaces_by_keyword(keywords: List[str]) -> List[dict]:
    """
    Use X API v2 'spaces' search endpoint to find live spaces matching keywords.
    Tweepy does not yet have a dedicated helper for every query; we can call the REST endpoint.
    """
    # Build query: search Spaces by query param 'query' referencing title/description
    # The X docs show the endpoint: GET /2/spaces/search
    # We'll use raw requests to the endpoint to be explicit.
    query_text = " OR ".join(keywords)
    url = "https://api.twitter.com/2/spaces/search"
    params = {
        "query": query_text,
        "state": "live",  # we want live spaces
        "space.fields": "id,title,host_ids,created_at,started_at,participant_count"
    }
    headers = {"Authorization": f"Bearer {BEARER_TOKEN}"}
    r = requests.get(url, headers=headers, params=params, timeout=15)
    r.raise_for_status()
    data = r.json()
    return data.get("data", [])

def matches_keywords(title: str, keywords: List[str]) -> bool:
    t = title.lower()
    for k in keywords:
        if k.lower() in t:
            return True
    return False

def tweet_about_space(space: dict):
    if not client_write:
        logger.info("No write client configured — would tweet: %s", space)
        return
    title = space.get("title", "<no-title>")
    space_id = space.get("id")
    hosts = space.get("host_ids", [])
    tweet_text = f"🔊 Live now: \"{title}\" — join here: https://twitter.com/i/spaces/{space_id}"
    try:
        resp = client_write.create_tweet(text=tweet_text)
        logger.info("Tweeted: %s", resp)
    except Exception as e:
        logger.exception("Failed to tweet: %s", e)

def main():
    seen: Set[str] = set()
    logger.info("Starting Spaces watcher (keywords=%s)", WATCH_KEYWORDS)

    while True:
        try:
            spaces = search_live_spaces_by_keyword(WATCH_KEYWORDS)
            for s in spaces:
                sid = s.get("id")
                title = s.get("title", "")
                if not sid:
                    continue
                if sid in seen:
                    continue
                if matches_keywords(title, WATCH_KEYWORDS):
                    logger.info("Found new matching Space: id=%s title=%s", sid, title)
                    tweet_about_space(s)
                    seen.add(sid)
            time.sleep(POLL_INTERVAL)
        except KeyboardInterrupt:
            logger.info("Stopping by user")
            break
        except Exception:
            logger.exception("Unexpected error — sleeping then retrying")
            time.sleep(30)

if __name__ == "__main__":
    main()
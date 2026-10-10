import os
import re
import html
import requests
import feedparser
from datetime import datetime, timedelta, timezone
from urllib.parse import quote
from google import genai
from googleapiclient.discovery import build

# ==========================================
# ENTERTAINMENT NEWS BOT
# ==========================================

TELEGRAM_BOT_TOKEN = os.environ["TELEGRAM_BOT_TOKEN"]
TELEGRAM_CHAT_ID = os.environ["TELEGRAM_CHAT_ID"]
GEMINI_API_KEY = os.environ["GEMINI_API_KEY"]
YOUTUBE_API_KEY = os.environ["YOUTUBE_API_KEY"]

GEMINI_MODEL = "gemini-2.5-flash"
MAX_NEWS = 12

gemini = genai.Client(api_key=GEMINI_API_KEY)

youtube = build(
    "youtube",
    "v3",
    developerKey=YOUTUBE_API_KEY,
    cache_discovery=False
)

HEADERS = {
    "User-Agent": "EntertainmentNewsBot/1.0"
}

# ==========================================
# TELEGRAM MESSAGE
# ==========================================

def send_telegram(text):
    url = (
        f"https://api.telegram.org/bot"
        f"{TELEGRAM_BOT_TOKEN}/sendMessage"
    )

    response = requests.post(
        url,
        data={
            "chat_id": TELEGRAM_CHAT_ID,
            "text": text,
            "disable_web_page_preview": True
        },
        timeout=30
    )

    response.raise_for_status()

    result = response.json()

    if not result.get("ok"):
        raise RuntimeError("Telegram message could not be sent.")

# ==========================================
# NEWS SOURCES
# ==========================================

NEWS_QUERIES = [
    (
        "Bollywood",
        "Bollywood film industry latest news"
    ),
    (
        "South Cinema",
        "Telugu Tamil Malayalam Kannada cinema latest news"
    ),
    (
        "Trailers",
        "new Indian movie trailer teaser released"
    ),
    (
        "OTT",
        "India OTT web series streaming latest news"
    ),
    (
        "Music",
        "Indian music industry Bollywood songs latest news"
    ),
    (
        "Box Office",
        "Indian movie box office collection latest"
    ),
]

def google_news_rss(category, query):
    url = (
        "https://news.google.com/rss/search?q="
        + quote(query + " when:3d")
        + "&hl=en-IN&gl=IN&ceid=IN:IN"
    )

    feed = feedparser.parse(url)
    items = []

    for entry in feed.entries[:8]:
        link = entry.get("link", "").strip()
        title = html.unescape(
            re.sub(r"<[^>]+>", "", entry.get("title", ""))
        ).strip()

        description = html.unescape(
            re.sub(
                r"<[^>]+>",
                " ",
                entry.get("summary", "")
            )
        ).strip()

        if not title or not link:
            continue

        items.append({
            "category": category,
            "title": title,
            "description": description[:700],
            "url": link,
            "source": "Google News RSS",
            "published": entry.get("published", "")
        })

    return items

# ==========================================
# YOUTUBE SEARCH
# ==========================================

YOUTUBE_QUERIES = [
    ("Bollywood", "Bollywood official trailer teaser"),
    ("South Cinema", "Telugu Tamil movie official trailer"),
    ("OTT", "Indian OTT web series official trailer"),
    ("Music", "Indian music label official song video"),
]

def search_youtube(category, query):
    items = []

    published_after = (
        datetime.now(timezone.utc) - timedelta(days=3)
    ).isoformat(timespec="seconds").replace("+00:00", "Z")

    result = youtube.search().list(
        part="snippet",
        q=query,
        type="video",
        order="date",
        publishedAfter=published_after,
        maxResults=5,
        regionCode="IN",
        relevanceLanguage="hi"
    ).execute()

    for video in result.get("items", []):
        video_id = video["id"].get("videoId")

        if not video_id:
            continue

        snippet = video.get("snippet", {})

        items.append({
            "category": category,
            "title": snippet.get("title", ""),
            "description": snippet.get("description", "")[:700],
            "url": f"https://www.youtube.com/watch?v={video_id}",
            "source": "YouTube",
            "published": snippet.get("publishedAt", ""),
            "channel": snippet.get("channelTitle", "")
        })

    return items

# ==========================================
# DUPLICATE FILTER
# ==========================================

def clean_text(text):
    return re.sub(r"\s+", " ", text or "").strip()

def unique_items(items):
    seen = set()
    unique = []

    for item in items:
        url = item.get("url", "").strip()

        if not url or url in seen:
            continue

        seen.add(url)
        unique.append(item)

    return unique

# ==========================================
# GEMINI HINDI SUMMARY
# ==========================================

def summarize_news(item):
    title = clean_text(item.get("title", ""))
    description = clean_text(item.get("description", ""))
    category = item.get("category", "")
    source = item.get("source", "")

    prompt = f"""
आप हिंदी मनोरंजन समाचार सहायक हैं।

नीचे दिए गए शीर्षक और विवरण के आधार पर ही जानकारी दें।
कोई नई घटना, तारीख, आँकड़ा या दावा अपनी तरफ से न जोड़ें।
अगर विवरण अधूरा है, तो केवल शीर्षक के आधार पर सीमित सारांश दें।
अफवाह को पक्की खबर न बताएं।
सारांश सरल, स्वाभाविक हिंदी में लिखें।

श्रेणी: {category}
स्रोत: {source}
शीर्षक: {title}
उपलब्ध विवरण: {description}

उत्तर का प्रारूप:
सारांश: 2 छोटे वाक्य।
वीडियो आइडिया: इस विषय पर एक उपयोगी YouTube वीडियो एंगल।

उत्तर संक्षिप्त रखें।
"""

    try:
        response = gemini.models.generate_content(
            model=GEMINI_MODEL,
            contents=prompt
        )

        summary = (response.text or "").strip()

        if summary:
            return summary

    except Exception as error:
        print(f"Gemini summary error: {error}")

    return (
        f"शीर्षक: {title}\n"
        "वीडियो आइडिया: इस खबर के तथ्य और आधिकारिक स्रोत की जाँच करें।"
    )

# ==========================================
# MAIN PROGRAM
# ==========================================

def main():
    print("Entertainment News Bot started.")

    all_items = []

    # Collect entertainment news RSS feeds
    for category, query in NEWS_QUERIES:
        try:
            items = google_news_rss(category, query)
            all_items.extend(items)
            print(f"{category}: {len(items)} RSS items")
        except Exception as error:
            print(f"RSS error ({category}): {error}")

    # Collect recent YouTube videos
    for category, query in YOUTUBE_QUERIES:
        try:
            items = search_youtube(category, query)
            all_items.extend(items)
            print(f"{category} YouTube: {len(items)} videos")
        except Exception as error:
            print(f"YouTube error ({category}): {error}")

    all_items = unique_items(all_items)

    if not all_items:
        send_telegram(
            "🎬 Entertainment News Bot\n\n"
            "इस बार कोई खबर नहीं मिल सकी। "
            "अगली बार फिर कोशिश की जाएगी।"
        )
        print("No items found.")
        return

    # Prefer the most recent items
    all_items.sort(
        key=lambda item: item.get("published", ""),
        reverse=True
    )

    selected_items = all_items[:MAX_NEWS]

    now = datetime.now().strftime("%d-%m-%Y %H:%M")

    send_telegram(
        "🎬 ENTERTAINMENT NEWS UPDATE\n"
        f"📅 {now}\n"
        f"📰 मिली खबरें/वीडियो: {len(selected_items)}\n\n"
        "नीचे खबरों के लिंक दिए जा रहे हैं।"
    )

    for index, item in enumerate(selected_items, start=1):
        title = clean_text(item.get("title", ""))
        category = item.get("category", "Entertainment")
        source = item.get("source", "")
        url = item.get("url", "")

        print(f"Summarizing {index}/{len(selected_items)}: {title}")

        summary = summarize_news(item)

        message = (
            f"🎬 {index}. {category}\n\n"
            f"📰 {title}\n\n"
            f"{summary}\n\n"
            f"🔎 स्रोत: {source}\n"
            f"🔗 लिंक: {url}"
        )

        # Telegram messages have a maximum length.
        if len(message) > 4000:
            message = message[:3900] + "\n\n[संदेश छोटा किया गया]"

        try:
            send_telegram(message)
        except Exception as error:
            print(f"Telegram send error: {error}")

    print("Entertainment News Bot finished.")

if __name__ == "__main__":
    main()

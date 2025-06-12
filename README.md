# craigslist-car-finder
import streamlit as st
import requests
from bs4 import BeautifulSoup
import pandas as pd

# Keywords
KEYWORDS_FAST = ["turbo", "v6", "v8", "manual", "sport", "coupe", "convertible"]
KEYWORDS_LOW_INSURANCE = ["civic", "corolla", "camry", "mazda", "accord", "elantra", "focus", "fit"]
GOOD_MPG_THRESHOLD = 25
MAX_PRICE = 9000

st.set_page_config(page_title="Craigslist Car Finder", layout="wide")
st.title("🚗 Craigslist Car Finder Under $9,000")

city = st.text_input("Enter Craigslist city code (e.g. kansascity, stlouis, dallas):", "kansascity")

if st.button("Search Cars"):

    BASE_URL = f"https://{city}.craigslist.org"
    SEARCH_URL = f"{BASE_URL}/search/cto?query=car&max_price={MAX_PRICE}&hasPic=1"

    try:
        response = requests.get(SEARCH_URL, timeout=10)
        soup = BeautifulSoup(response.text, 'html.parser')
        listings = soup.find_all('li', class_='result-row')
        results = []

        for listing in listings:
            title_tag = listing.find('a', class_='result-title')
            if not title_tag:
                continue

            title = title_tag.text.strip()
            link = title_tag['href']
            price_tag = listing.find('span', class_='result-price')
            price = int(price_tag.text.replace("$", "").replace(",", "")) if price_tag else 0
            date = listing.find('time')['datetime'] if listing.find('time') else "Unknown"

            fast_score = sum(1 for word in KEYWORDS_FAST if word in title.lower())
            low_insurance = any(word in title.lower() for word in KEYWORDS_LOW_INSURANCE)
            est_mpg = 30 if low_insurance else 20

            if price <= MAX_PRICE and fast_score > 0 and est_mpg >= GOOD_MPG_THRESHOLD and low_insurance:
                results.append({
                    "Title": title,
                    "Price": price,
                    "Fast Score": fast_score,
                    "MPG": est_mpg,
                    "Date": date,
                    "Link": f"[View Listing]({link})"
                })

        if results:
            df = pd.DataFrame(results).sort_values(by=["Fast Score", "MPG"], ascending=False)
            st.write(df.to_markdown(index=False), unsafe_allow_html=True)
        else:
            st.warning("No matching cars found. Try a different city or adjust your filters.")
    except Exception as e:
        st.error(f"Error: {e}")

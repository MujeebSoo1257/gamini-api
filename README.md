

import streamlit as st
import pandas as pd
from textblob import TextBlob
import plotly.express as px
from wordcloud import WordCloud
import matplotlib.pyplot as plt
import re
from collections import Counter

# ----------------------------
# PAGE CONFIG
# ----------------------------
st.set_page_config(
    page_title="Sentiment Analysis App",
    page_icon="😊",
    layout="wide"
)

st.title("😊 Sentiment Analysis App")

# ----------------------------
# SESSION STATE
# ----------------------------
if "history" not in st.session_state:
    st.session_state.history = []

# ----------------------------
# FUNCTIONS
# ----------------------------
def analyze_sentiment(text):
    blob = TextBlob(text)
    polarity = blob.sentiment.polarity
    subjectivity = blob.sentiment.subjectivity

    if polarity > 0.05:
        sentiment = "Positive"
    elif polarity < -0.05:
        sentiment = "Negative"
    else:
        sentiment = "Neutral"

    return sentiment, polarity, subjectivity


def extract_keywords(text):
    words = re.findall(r'\b[a-zA-Z]{3,}\b', text.lower())
    stop_words = {'the','and','for','this','that','with','you','are','was'}
    words = [w for w in words if w not in stop_words]
    return Counter(words).most_common(10)

# ----------------------------
# SIDEBAR
# ----------------------------
mode = st.sidebar.radio("Select Mode", [
    "Single Text",
    "Batch Analysis",
    "Demo"
])

# ----------------------------
# SINGLE TEXT
# ----------------------------
if mode == "Single Text":
    text = st.text_area("Enter Text")

    if st.button("Analyze"):
        if text.strip():
            sentiment, polarity, subjectivity = analyze_sentiment(text)

            st.success(f"Sentiment: {sentiment}")
            st.write(f"Polarity: {polarity:.3f}")
            st.write(f"Subjectivity: {subjectivity:.3f}")

            # Save history
            st.session_state.history.append({
                "text": text,
                "sentiment": sentiment,
                "polarity": polarity,
                "subjectivity": subjectivity
            })

            # Keywords
            keywords = extract_keywords(text)
            if keywords:
                df = pd.DataFrame(keywords, columns=["Word","Count"])
                fig = px.bar(df, x="Count", y="Word", orientation="h")
                st.plotly_chart(fig)

            # Wordcloud
            wc = WordCloud().generate(text)
            fig, ax = plt.subplots()
            ax.imshow(wc)
            ax.axis("off")
            st.pyplot(fig)

        else:
            st.warning("Enter text first!")

    # History
    if st.session_state.history:
        st.subheader("History")
        st.dataframe(pd.DataFrame(st.session_state.history))

# ----------------------------
# BATCH MODE
# ----------------------------
elif mode == "Batch Analysis":
    file = st.file_uploader("Upload CSV or Excel", type=["csv","xlsx"])

    if file:
        try:
            if file.name.endswith(".csv"):
                df = pd.read_csv(file)
            else:
                df = pd.read_excel(file)
        except Exception as e:
            st.error(f"File error: {e}")
            st.stop()

        st.dataframe(df.head())

        column = st.selectbox("Select Text Column", df.columns)

        if st.button("Analyze All"):
            sentiments = []
            polarity_list = []
            subjectivity_list = []

            for txt in df[column].astype(str):
                s, p, sub = analyze_sentiment(txt)
                sentiments.append(s)
                polarity_list.append(p)
                subjectivity_list.append(sub)

            df["Sentiment"] = sentiments
            df["Polarity"] = polarity_list
            df["Subjectivity"] = subjectivity_list

            st.success("Done!")

            st.dataframe(df)

            fig = px.pie(df, names="Sentiment", title="Sentiment Distribution")
            st.plotly_chart(fig)

            st.download_button(
                "Download CSV",
                df.to_csv(index=False),
                "results.csv"
            )

# ----------------------------
# DEMO MODE
# ----------------------------
else:
    samples = [
        "I love this product!",
        "This is the worst experience ever.",
        "It is okay, nothing special."
    ]

    for txt in samples:
        if st.button(txt):
            s, p, sub = analyze_sentiment(txt)
            st.info(f"{txt} → {s} (Polarity: {p:.2f})")

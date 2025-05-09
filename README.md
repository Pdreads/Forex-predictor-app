# Forex-predictor-app
Prediction made so you can make the best trades 
import streamlit as st
import yfinance as yf
import pandas as pd
import pandas_ta as ta
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report

# --- UI ---
st.title("Forex Signal Predictor")
st.markdown("Analyze historical forex data and get a Buy/Sell/Hold signal.")

# User input
pair = st.text_input("Forex Pair (e.g., EURUSD=X)", "EURUSD=X")
period = st.selectbox("How much history?", ["1wk", "1mo", "3mo", "6mo", "1y", "2y"], index=3)
interval = st.selectbox("Data Interval", ["1h", "4h", "1d"], index=0)

if st.button("Analyze Pair"):

    st.info("Fetching data...")
    df = yf.download(pair, period=period, interval=interval)

    if df.empty:
        st.error("No data found. Please check the pair or time range.")
    else:
        # Technical Indicators
        df["SMA_20"] = df["Close"].rolling(window=20).mean()
        df["RSI"] = ta.rsi(df["Close"], length=14)
        macd = ta.macd(df["Close"])
        df["MACD"] = macd["MACD_12_26_9"]
        df["Signal"] = macd["MACDs_12_26_9"]
        bb = ta.bbands(df["Close"])
        df["BB_Upper"] = bb["BBU_20_2.0"]
        df["BB_Lower"] = bb["BBL_20_2.0"]

        df = df.dropna()

        # Create labels
        df["Future_Close"] = df["Close"].shift(-1)
        df["Signal_Label"] = df.apply(lambda row:
            1 if row["Future_Close"] > row["Close"] * 1.002 else
            -1 if row["Future_Close"] < row["Close"] * 0.998 else 0, axis=1)

        X = df[["Close", "SMA_20", "RSI", "MACD", "Signal", "BB_Upper", "BB_Lower"]]
        y = df["Signal_Label"]

        # Train/test split
        X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2, shuffle=False)

        model = RandomForestClassifier(n_estimators=100)
        model.fit(X_train, y_train)

        preds = model.predict(X_test)
        report = classification_report(y_test, preds, output_dict=True)

        # Predict the latest row
        latest_data = df.iloc[[-1]][X.columns]
        action = model.predict(latest_data)[0]

        st.subheader("Recommendation")
        if action == 1:
            st.success("→ Buy")
        elif action == -1:
            st.error("→ Sell")
        else:
            st.warning("→ Hold")

        st.subheader("Model Stats")
        st.json(report)
        st.dataframe(df.tail(5))

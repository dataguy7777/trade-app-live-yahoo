import streamlit as st
import pandas as pd
import yfinance as yf
import plotly.express as px
import logging
from datetime import datetime
from typing import List

# -------------------------------------------------
# Logging configuration
# -------------------------------------------------
logging.basicConfig(
    format="%(asctime)s - %(levelname)s - %(message)s",
    level=logging.INFO,
    datefmt="%Y-%m-%d %H:%M:%S",
)
logger = logging.getLogger(__name__)

# -------------------------------------------------
# Data layer
# -------------------------------------------------
@st.cache_data(show_spinner=False)
def fetch_hourly_data(tickers: List[str], period: str = "5d") -> pd.DataFrame:
    """Download hourly OHLCV data for the given tickers from Yahoo Finance.

    Args:
        tickers (List[str]): List of ticker symbols. Example: ["QQQ", "QQQ3"].
        period (str, optional): Window of historical data recognised by Yahoo Finance. Defaults to "5d".
            Example values: "1d", "5d", "1mo", "3mo", "6mo", "1y".

    Returns:
        pd.DataFrame: Multi‑index (Datetime → Ticker) DataFrame with OHLCV columns.

    Example:
        >>> df = fetch_hourly_data(["QQQ"], "7d")
        >>> print(df.head())
    """
    logger.info("Fetching data for %s | period=%s", ", ".join(tickers), period)

    # yf.download supports space‑separated ticker strings
    raw = yf.download(
        tickers=" ".join(tickers),
        period=period,
        interval="60m",  # hourly granularity
        group_by="ticker",
        auto_adjust=True,
        threads=True,
        progress=False,
    )

    # When only one ticker is supplied, yfinance drops the first level
    if len(tickers) == 1:
        raw.columns = pd.MultiIndex.from_product([tickers, raw.columns])

    # Normalise timezone to the user's locale (Europe/Rome)
    raw.index = raw.index.tz_localize("UTC").tz_convert("Europe/Rome")

    logger.info("Successfully fetched %d rows", len(raw))
    return raw

# -------------------------------------------------
# UI layer
# -------------------------------------------------

def main() -> None:
    """Streamlit app entry‑point."""
    st.set_page_config(page_title="Hourly ETF Dashboard", layout="wide")

    st.title("📈 Hourly ETF Dashboard: QQQ & QQQ3")
    st.markdown(
        """
        Visualise **hourly** price movements of Invesco NASDAQ‑100 ETFs using
        real‑time data from Yahoo Finance (`yfinance`).
        """
    )

    # ---------- Sidebar controls ----------
    st.sidebar.header("⚙️ Parameters")
    default_tickers = ["QQQ", "QQQ3"]
    tickers = st.sidebar.multiselect(
        "Select ticker symbols", default_tickers, default=default_tickers
    )
    period = st.sidebar.selectbox(
        "Historical period", ["1d", "5d", "10d", "30d", "60d", "90d"], index=1
    )

    fetch_btn = st.sidebar.button("Fetch data", type="primary")

    st.sidebar.markdown("---")
    st.sidebar.caption(
        "Data fetched live via **yfinance**. Hourly interval is limited to the last 730 hours (~30 days)."
    )

    # ---------- Main area ----------
    if fetch_btn:
        if not tickers:
            st.error("Please select at least one ticker symbol.")
            st.stop()

        with st.spinner("Downloading data …"):
            df = fetch_hourly_data(tickers, period)

        st.success("Data downloaded ✅")
        st.dataframe(df.head(), use_container_width=True)

        # Plot each ticker separately for clarity
        for ticker in tickers:
            close_series = df[ticker]["Close"]
            fig = px.line(
                close_series,
                labels={"value": "Price (USD)", "index": "Datetime"},
                title=f"Hourly Close Price – {ticker}",
            )
            st.plotly_chart(fig, use_container_width=True)

        # Last refreshed timestamp
        st.caption(
            f":clock1: Data last refreshed at {datetime.now().astimezone().strftime('%Y-%m-%d %H:%M:%S %Z')}"
        )


if __name__ == "__main__":
    main()

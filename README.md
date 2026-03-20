import streamlit as st
import pandas as pd
from google.ads.googleads.client import GoogleAdsClient
import io

st.set_page_config(page_title="Keyword Planner Admin", layout="wide")

# Sidebar: Konfiguration
st.sidebar.header("API-Konfig")
CUSTOMER_ID = st.sidebar.text_input("Google Ads Customer ID", type="password")
DEVELOPER_TOKEN = st.sidebar.text_input("Developer Token", type="password")
SEED_KEYWORDS = st.sidebar.text_area("Seed-Keywords (z.B. hypnose ausbildung)", "hypnose ausbildung")

if st.sidebar.button("API-Verbinden"):
    try:
        client = GoogleAdsClient.load_from_dict({
            "developer_token": DEVELOPER_TOKEN,
            "customer_id": CUSTOMER_ID,
            "use_proto_plus": True,
        })
        st.sidebar.success("✅ API verbunden!")
        st.session_state.client = client
    except Exception as e:
        st.sidebar.error(f"❌ Fehler: {e}")

# Hauptbereich
st.title("🚀 Auto-Keyword Planner Admin Dashboard")
st.info("**Automatisiert**: Keywords abrufen → CSV-Export → Filter → Prompt generieren → In Tools übertragen")

if "client" in st.session_state and st.button("🔄 Keywords abrufen"):
    with st.spinner("Hole Daten vom Keyword Planner..."):
        kp_service = st.session_state.client.get_service("KeywordPlanIdeaService")
        request = st.session_state.client.get_type("GenerateKeywordIdeasRequest")
        request.customer_id = CUSTOMER_ID
        request.language = st.session_state.client.get_service("LanguageService").language_constant("de")
        request.geo_target_constants.append(st.session_state.client.get_service("GeoTargetService").geo_target_constant("DE"))
        request.keyword_seed.keywords.extend([kw.strip() for kw in SEED_KEYWORDS.split(",")])
        
        response = kp_service.generate_keyword_ideas(request=request)
        data = []
        for row in response.results[:100]:  # Top 100
            data.append({
                "Keyword": row.text,
                "Monatl. Suchen": row.keyword_idea_metrics.avg_monthly_searches,
                "Wettbewerb": row.keyword_idea_metrics.competition.name,
                "Top CPC": row.keyword_idea_metrics.max.cpc_micros / 1_000_000
            })
        
        df = pd.DataFrame(data)
        st.session_state.df = df[df["Monatl. Suchen"] > 0].sort_values("Monatl. Suchen", ascending=False)
    
    if "df" in st.session_state:
        st.success(f"✅ {len(st.session_state.df)} Keywords gefunden!")
        st.dataframe(st.session_state.df, use_container_width=True)

# Filter & Analyse
if "df" in st.session_state:
    col1, col2 = st.columns(2)
    with col1:
        min_searches = st.slider("Min. Suchvolumen", 0, 10000, 100)
        filtered_df = st.session_state.df[st.session_state.df["Monatl. Suchen"] >= min_searches]
    
    with col2:
        top_n = st.slider("Top N Keywords", 5, 50, 10)
        top_keywords = filtered_df.head(top_n)["Keyword"].tolist()
    
    st.subheader("📊 Top Keywords")
    st.write("|".join(top_keywords))

# Export & Übertragung
    col1, col2, col3 = st.columns(3)
    with col1:
        csv = filtered_df.to_csv(index=False).encode('utf-8')
        st.download_button("📥 CSV Download", csv, "keywords.csv", "text/csv")
    
    with col2:
        if st.button("➡️ Google Sheets pushen"):
            # Hier Google Sheets API integrieren (gspread)
            st.info("Sheets-Link: [Öffnen](https://sheets.google.com) – CSV kopieren")
    
    with col3:
        prompt = f"Analysiere Keywords: {', '.join(top_keywords[:5])}. Beste Hypnose-Ausbilder finden."
        st.text_area("🤖 Auto-Prompt", prompt, height=100)

# Dashboard-Übersicht
if "df" in st.session_state:
    col1, col2 = st.columns(2)
    with col1: st.metric("Gesamt Keywords", len(st.session_state.df))
    with col2: st.metric("Ø Suchvolumen", st.session_state.df["Monatl. Suchen"].mean())

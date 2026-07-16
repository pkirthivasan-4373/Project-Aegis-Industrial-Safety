import streamlit as st
import firebase_admin
from firebase_admin import credentials
from firebase_admin import db

# --- 1. Page Configuration ---
st.set_page_config(page_title="Aegis Command Center", layout="wide")
st.title("🛡️ Project Aegis: Fleet Manager Dashboard")
st.markdown("Real-time worker safety and environmental monitoring.")
st.markdown("---")

# --- 2. Firebase Authentication ---
# We check if it's already initialized so Streamlit doesn't crash on reload
if not firebase_admin._apps:
    # This points to the JSON key you dragged into the folder
    cred = credentials.Certificate("firebase_credentials.json")
    firebase_admin.initialize_app(cred, {
        'databaseURL': 'https://project-aegis-fd75b-default-rtdb.asia-southeast1.firebasedatabase.app/'
    })

# --- 3. UI Layout ---
col1, col2 = st.columns(2)

with col1:
    st.subheader("👨‍🔧 Worker Orientation")
    orientation_alert = st.empty()
    z_metric = st.empty()

with col2:
    st.subheader("💨 Air Quality (Gas Level)")
    gas_alert = st.empty()
    gas_metric = st.empty()

st.markdown("---")
st.caption("Auto-refreshing dashboard powered by Streamlit & Google Firebase.")

# --- 4. Live Data Fetching ---
if st.button("🔄 Fetch Live Data"):
    with st.spinner("Pinging Google Cloud..."):
        # Target the specific branch where your ESP32 is saving data
        ref = db.reference('/node1')
        data = ref.get()

        if data:
            gas_value = data.get('gas', 0)
            z_axis = data.get('accelZ', 0.0)

            # --- Logic: Fall Detection ---
            z_metric.metric("Z-Axis Gravity", f"{z_axis:.2f} m/s²")
            if z_axis < 0:
                orientation_alert.error("🚨 CRITICAL: Fall Detected! Worker orientation inverted.")
            else:
                orientation_alert.success("✅ Worker is Upright and Safe.")

            # --- Logic: Gas Detection ---
            gas_metric.metric("Raw Gas Value", gas_value)
            # Adjust this 3500 threshold based on your actual room conditions
            if gas_value > 3500: 
                gas_alert.error("⚠️ HAZARD: High Gas Concentration Detected!")
            else:
                gas_alert.success("✅ Air Quality Normal.")
        else:
            st.warning("No data found in Firebase. Is the ESP32 plugged in?")

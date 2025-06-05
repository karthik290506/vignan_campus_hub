import streamlit as st
import pandas as pd
from datetime import datetime
import json
import os
import requests
from io import BytesIO
import base64

st.set_page_config(layout="wide", page_title="Vignan Campus Hub", page_icon="🏫")

st.markdown(
    """
    <style>
    :root {
        --primary: #62aeff;
        --secondary: #3a3b47;
        --accent: #ff4d4d;
        --text: #eee;
        --bg: #262730;
    }
    body {
        color: var(--text);
        background-color: var(--bg);
    }
    .sidebar .sidebar-content {
        background-color: #222 !important;
    }
    .stHeader {
        background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
        color: white;
        padding: 1.5rem;
        border-radius: 10px;
        box-shadow: 0 4px 6px rgba(0,0,0,0.1);
        margin-bottom: 2rem;
    }
    .stSubheader {
        color: var(--primary) !important;
        border-bottom: 2px solid var(--primary);
        padding-bottom: 0.5rem;
        margin-top: 1.5rem !important;
    }
    .card {
        background: var(--secondary);
        border-radius: 10px;
        padding: 1.5rem;
        margin-bottom: 1rem;
        box-shadow: 0 2px 4px rgba(0,0,0,0.1);
        transition: transform 0.2s;
    }
    .card:hover {
        transform: translateY(-5px);
    }
    .emergency-card {
        background: #542e2e;
        border-left: 5px solid var(--accent);
        animation: pulse 2s infinite;
    }
    @keyframes pulse {
        0% { box-shadow: 0 0 0 0 rgba(255,77,77,0.4); }
        70% { box-shadow: 0 0 0 10px rgba(255,77,77,0); }
        100% { box-shadow: 0 0 0 0 rgba(255,77,77,0); }
    }
    .event-card {
        border-left: 5px solid var(--primary);
    }
    .resource-card {
        border-left: 5px solid #4CAF50;
    }
    .status-badge {
        display: inline-block;
        padding: 0.25rem 0.5rem;
        border-radius: 20px;
        font-size: 0.75rem;
        font-weight: bold;
        margin-left: 0.5rem;
    }
    .upcoming { background: #4CAF50; }
    .ongoing { background: #FFC107; color: #333; }
    .completed { background: #F44336; }
    .weather-widget {
        background: linear-gradient(135deg, #1e3c72 0%, #2a5298 100%);
        border-radius: 10px;
        padding: 1rem;
        margin-bottom: 1rem;
    }
    .admin-section {
        background: rgba(255,255,255,0.1);
        padding: 1rem;
        border-radius: 10px;
        margin-bottom: 1rem;
    }
    .timetable {
        width: 100%;
        border-collapse: collapse;
    }
    .timetable th, .timetable td {
        border: 1px solid #444;
        padding: 8px;
        text-align: center;
    }
    .timetable th {
        background-color: var(--primary);
        color: white;
    }
    .timetable tr:nth-child(even) {
        background-color: #333;
    }
    </style>
    """,
    unsafe_allow_html=True,
)

DATA_FILE = "campus_data.json"
ADMIN_PASSWORD = "campusadmin123"

def load_data():
    default_data = {
        "contacts": {
            "columns": ["Department", "Phone", "Email", "Location"],
            "data": [
                ["Admissions Office", "123-456-7890", "admissions@vignan.edu", "Admin Block"],
                ["Library", "987-654-3210", "library@vignan.edu", "Central Library"]
            ]
        },
        "events": {
            "columns": ["Date", "Title", "Location", "Time", "Category"],
            "data": [
                ["2025-06-01", "Career Fair", "Auditorium", "10:00 AM", "Career"],
                ["2025-06-15", "Tech Workshop", "Lab 3", "2:00 PM", "Technical"]
            ]
        },
        "emergency": {
            "columns": ["Contact", "Number", "Available", "Location"],
            "data": [
                ["Campus Security", "9944332211", "24/7", "Security Office"],
                ["Medical", "9988776655", "24/7", "Sick Room"]
            ]
        },
        "transport": {
            "bus_routes": [
                ["Route 1", "Main Gate - Admin Block - Hostels", "7:00 AM, 1:00 PM, 5:00 PM"],
                ["Route 2", "North Campus - Library - Canteen", "Every 30 minutes from 8 AM to 8 PM"],
                ["Route 3", "Hostels - Sports Complex - Main Gate", "6:00 AM, 4:00 PM, 9:00 PM"]
            ],
            "shuttle_times": [
                ["Library to Hostels", "12:20 PM - 12:25 PM", "2 Busses at 12:20"],
                ["Library to Hostels", "1:10 PM - 1:15 PM", "2 Busses at 1:10"],
                ["Hostels to Library", "12:30 PM - 12:55 PM", "2 Busses at 12:20"],
                ["Hostels to Library", "1:20 PM - 1:45 PM", "2 Busses at 12:20"]
            ]
        },
        "food": {
            "canteens": [
                ["Main Canteen", "7:00 AM - 5:00 PM", "Near Civil Engg Lab", "Veg & Non-Veg"],
                ["Cafeteria", "8:00 AM - 5:00 PM", "Near Library", "Fast Food"],
            ],
            "hostels": [
                ["Boys Hostel A", "150", "Single/Double", "AC/Non-AC"],
                ["Girls Hostel B", "100", "Triple", "Non-AC"],
                ["New Hostel", "50", "Single/Double/triple", "AC"]
            ]
        },
        "lost_found": [
            ["ID Card", "Library", "2025-05-10", "Not claimed"],
            ["Laptop Charger", "Lab 3", "2025-05-12", "Not claimed"],
            ["Water Bottle", "Auditorium", "2025-05-14", "Claimed"]
        ],
        "academic_pdfs": []
    }
    try:
        with open(DATA_FILE, "r") as f:
            loaded_data = json.load(f)
            for key in default_data.keys():
                if key not in loaded_data:
                    loaded_data[key] = default_data[key]
            return loaded_data
    except (FileNotFoundError, json.JSONDecodeError):
        return default_data

def save_data(data):
    with open(DATA_FILE, "w") as f:
        json.dump(data, f)

@st.cache_data(ttl=3600)
def get_weather(latitude=17.3331139, longitude=78.718202):
    try:
        url = f"https://api.open-meteo.com/v1/forecast?latitude={latitude}&longitude={longitude}&current_weather=true&hourly=temperature_2m,relativehumidity_2m,windspeed_10m&timezone=auto"
        response = requests.get(url)
        response.raise_for_status()
        weather_data = response.json()
        current = weather_data.get('current_weather', {})
        hourly = weather_data.get('hourly', {})
        return {
            'current_temp': current.get('temperature'),
            'current_windspeed': current.get('windspeed'),
            'current_weathercode': current.get('weathercode'),
            'hourly_temps': hourly.get('temperature_2m', []),
            'hourly_times': hourly.get('time', [])
        }
    except Exception as e:
        print(f"Error fetching weather: {e}")
        return None

def get_weather_description(code):
    weather_codes = {
        0: "Clear sky",
        1: "Mainly clear",
        2: "Partly cloudy",
        3: "Overcast",
        45: "Fog",
        48: "Depositing rime fog",
        51: "Light drizzle",
        53: "Moderate drizzle",
        55: "Dense drizzle",
        56: "Light freezing drizzle",
        57: "Dense freezing drizzle",
        61: "Slight rain",
        63: "Moderate rain",
        65: "Heavy rain",
        66: "Light freezing rain",
        67: "Heavy freezing rain",
        71: "Slight snow fall",
        73: "Moderate snow fall",
        75: "Heavy snow fall",
        77: "Snow grains",
        80: "Slight rain showers",
        81: "Moderate rain showers",
        82: "Violent rain showers",
        85: "Slight snow showers",
        86: "Heavy snow showers",
        95: "Thunderstorm",
        96: "Thunderstorm with slight hail",
        99: "Thunderstorm with heavy hail"
    }
    return weather_codes.get(code, "Unknown weather")

def authenticate():
    if 'authenticated' not in st.session_state:
        st.session_state.authenticated = False
    if not st.session_state.authenticated:
        with st.sidebar:
            st.markdown("---")
            st.subheader("Admin Login")
            password = st.text_input("Enter Admin Password", type="password")
            if st.button("Login"):
                if password == ADMIN_PASSWORD:
                    st.session_state.authenticated = True
                    st.success("Logged in successfully!")
                else:
                    st.error("Incorrect password")
    return st.session_state.authenticated

def clean_dataframe(df):
    df = df.replace('', None)
    df = df.dropna(how='all')
    df = df.reset_index(drop=True)
    return df

def show_admin_panel(data):
    st.sidebar.markdown("---")
    st.sidebar.subheader("Admin Panel")
    category = st.sidebar.selectbox("Select Data to Edit",
                                  ["Contacts", "Events", "Emergency", "Transport", "Food", "Lost & Found"])
    st.markdown("---")
    st.subheader(f"Editing {category}")
   
    if category == "Contacts":
        df = pd.DataFrame(data["contacts"]["data"], columns=data["contacts"]["columns"])
    elif category == "Events":
        df = pd.DataFrame(data["events"]["data"], columns=data["events"]["columns"])
    elif category == "Emergency":
        df = pd.DataFrame(data["emergency"]["data"], columns=data["emergency"]["columns"])
    elif category == "Transport":
        df = pd.DataFrame(data["transport"]["bus_routes"], columns=["Route", "Stops", "Timings"])
    elif category == "Food":
        df = pd.DataFrame(data["food"]["canteens"], columns=["Name", "Timings", "Location", "Menu"])
    else:
        df = pd.DataFrame(data["lost_found"], columns=["Item", "Location Found", "Date", "Status"])
   
    edited_df = st.data_editor(df, num_rows="dynamic", use_container_width=True)
   
    if st.button("Save Changes"):
        edited_df = clean_dataframe(edited_df)
       
        if category == "Contacts":
            data["contacts"]["data"] = edited_df.values.tolist()
        elif category == "Events":
            data["events"]["data"] = edited_df.values.tolist()
        elif category == "Emergency":
            data["emergency"]["data"] = edited_df.values.tolist()
        elif category == "Transport":
            data["transport"]["bus_routes"] = edited_df.values.tolist()
        elif category == "Food":
            data["food"]["canteens"] = edited_df.values.tolist()
        else:
            data["lost_found"] = edited_df.values.tolist()
       
        save_data(data)
        st.success("Changes saved successfully!")
        st.rerun()

def show_header():
    st.markdown("""
    <div class='stHeader'>
        <center>
            <h1>VIGNAN INSTITUTE OF TECHNOLOGY AND SCIENCE</h1>
            <h2>🏫 Campus Resource Hub</h2>
            <p>Your one-stop portal for all campus resources and information</p>
        </center>
    </div>
    """, unsafe_allow_html=True)

def show_sidebar():
    with st.sidebar:
        st.title("Navigate Campus")
        page = st.radio(
            "Main Menu",
            ["🏠 Dashboard", "📞 Directory", "📅 Events", "📚 Academics", "🚌 Transport",
             "🍔 Food & Stay", "🚨 Emergency", "🔍 Lost & Found"],
            label_visibility="collapsed"
        )
        weather_data = get_weather()
        if weather_data:
            current_temp = weather_data['current_temp']
            weather_desc = get_weather_description(weather_data['current_weathercode'])
            st.markdown(f"""
            <div class='weather-widget'>
                <h4>🌤️ Campus Weather</h4>
                <p>Now: {weather_desc}, {current_temp}°C</p>
                <p>Wind: {weather_data['current_windspeed']} km/h</p>
            </div>
            """, unsafe_allow_html=True)
        else:
            st.markdown("""
            <div class='weather-widget'>
                <h4>🌤️ Campus Weather</h4>
                <p>Weather data unavailable</p>
            </div>
            """, unsafe_allow_html=True)
        st.markdown("""
        <div style="margin-top: 2rem;">
            <h4>Quick Contacts</h4>
            <p>📞 Security: <strong>9988776655</strong></p>
            <p>🏥 Medical: <strong>944332211</strong></p>
            <p>📧 Helpdesk: <strong>help@vignan.edu</strong></p>
        </div>
        """, unsafe_allow_html=True)
    return page

def show_dashboard(data):
    st.markdown("<div class='stSubheader'><h2>📊 Campus Dashboard</h2></div>", unsafe_allow_html=True)
    col1, col2, col3 = st.columns(3)
    with col1:
        upcoming_events = 0
        next_event = "None scheduled"
        for e in data["events"]["data"]:
            if e and e[0]:
                try:
                    event_date = datetime.strptime(e[0], "%Y-%m-%d").date()
                    if event_date >= datetime.now().date():
                        upcoming_events += 1
                        if next_event == "None scheduled":
                            next_event = e[1]
                except (ValueError, TypeError):
                    continue
        st.markdown(f"""
        <div class='card'>
            <h3>📅 Upcoming Events</h3>
            <h1 style="color: var(--primary);">{upcoming_events}</h1>
            <p>Next event: {next_event}</p>
        </div>
        """, unsafe_allow_html=True)
    with col2:
        st.markdown("""
        <div class='card'>
            <h3>📚 Academic Resources</h3>
            <h1 style="color: #4CAF50;">6</h1>
            <p>Available online platforms</p>
        </div>
        """, unsafe_allow_html=True)
    with col3:
        st.markdown("""
        <div class='card'>
            <h3>🔍 Lost Items</h3>
            <h1 style="color: var(--accent);">3</h1>
            <p>Waiting to be claimed</p>
        </div>
        """, unsafe_allow_html=True)
    st.markdown("<div class='stSubheader'><h3>📢 Recent Announcements</h3></div>", unsafe_allow_html=True)
    announcements = [
        {"title": "Library Extended Hours", "date": "2025-05-14", "content": "Library will remain open until 5PM during week days."},
        {"title": "Parking Restrictions", "date": "2025-05-12", "content": "No parking in Parking area from June 5-20 due to construction."},
        {"title": "Scholarship Applications", "date": "2025-05-10", "content": "Deadline for scholarship applications extended to June 30."}
    ]
    for announcement in announcements:
        st.markdown(f"""
        <div class='card'>
            <h4>{announcement['title']}</h4>
            <p><small>{announcement['date']}</small></p>
            <p>{announcement['content']}</p>
        </div>
        """, unsafe_allow_html=True)

def show_directory(data):
    st.markdown("<div class='stSubheader'><h2>📞 Campus Directory</h2></div>", unsafe_allow_html=True)
    search_query = st.text_input("Search departments or services...", key="directory_search")
    df = pd.DataFrame(data["contacts"]["data"], columns=data["contacts"]["columns"])
    if search_query:
        df = df[df.apply(lambda row: row.astype(str).str.contains(search_query, case=False).any(), axis=1)]
    st.dataframe(df, use_container_width=True, hide_index=True)

def show_events(data):
    st.markdown("<div class='stSubheader'><h2>📅 Campus Events</h2></div>", unsafe_allow_html=True)
    col1, col2 = st.columns(2)
    with col1:
        categories = list(set([e[4] for e in data["events"]["data"] if len(e) > 4]))
        event_category = st.selectbox("Filter by category", ["All"] + categories)
    with col2:
        date_range = st.selectbox("Filter by date", ["Upcoming", "This week", "This month", "All"])
    events_list = []
    for e in data["events"]["data"]:
        if len(e) >= 5:
            try:
                date = datetime.strptime(e[0], "%Y-%m-%d") if e[0] else None
                if date:
                    events_list.append({
                        "Date": date,
                        "Title": e[1],
                        "Location": e[2],
                        "Time": e[3],
                        "Category": e[4]
                    })
            except (ValueError, TypeError):
                continue
    events_df = pd.DataFrame(events_list)
    if not events_df.empty:
        if event_category != "All":
            events_df = events_df[events_df['Category'] == event_category]
        today = datetime.now().date()
        if date_range == "This week":
            week_end = today + pd.Timedelta(days=7)
            events_df = events_df[(events_df['Date'].dt.date >= today) & (events_df['Date'].dt.date <= week_end)]
        elif date_range == "This month":
            month_end = today + pd.Timedelta(days=30)
            events_df = events_df[(events_df['Date'].dt.date >= today) & (events_df['Date'].dt.date <= month_end)]
        elif date_range == "Upcoming":
            events_df = events_df[events_df['Date'].dt.date >= today]
    if events_df.empty:
        st.info("No events match your filters.")
    else:
        for _, event in events_df.iterrows():
            status = "upcoming" if event['Date'] > datetime.now() else "ongoing" if event['Date'].date() == today else "completed"
            status_color = {"upcoming": "upcoming", "ongoing": "ongoing", "completed": "completed"}[status]
            st.markdown(f"""
            <div class='card event-card'>
                <div style="display: flex; justify-content: space-between; align-items: center;">
                    <h3>{event['Title']}</h3>
                    <span class='status-badge {status_color}'>{status.capitalize()}</span>
                </div>
                <p>🗓️ <strong>Date:</strong> {event['Date'].strftime('%A, %B %d, %Y')}</p>
                <p>🕒 <strong>Time:</strong> {event['Time']}</p>
                <p>📍 <strong>Location:</strong> {event['Location']}</p>
                <p>🏷️ <strong>Category:</strong> {event['Category']}</p>
            </div>
            """, unsafe_allow_html=True)

def show_academics(data):
    st.markdown("<div class='stSubheader'><h2>📚 Academics</h2></div>", unsafe_allow_html=True)
    if st.session_state.get('authenticated', False):
        st.markdown("<div class='stSubheader'><h3>📤 Upload Study Materials (Admin Only)</h3></div>", unsafe_allow_html=True)
        with st.expander("Upload PDF Document"):
            with st.form("pdf_upload_form"):
                pdf_file = st.file_uploader("Choose a PDF file", type="pdf")
                subject = st.text_input("Subject Name")
                description = st.text_area("Description")
                semester = st.selectbox("Semester", ["1st", "2nd", "3rd", "4th", "5th", "6th", "7th", "8th"])
                if st.form_submit_button("Upload PDF"):
                    if pdf_file is not None:
                        existing_pdf = next((pdf for pdf in data.get("academic_pdfs", []) 
                                          if pdf["subject"].lower() == subject.lower() 
                                          and pdf["semester"] == semester), None)
                        if existing_pdf:
                            st.error(f"A PDF for {subject} ({semester} semester) already exists!")
                        else:
                            os.makedirs("academics", exist_ok=True)
                            file_path = os.path.join("academics", f"{subject}_{semester}.pdf")
                            with open(file_path, "wb") as f:
                                f.write(pdf_file.getbuffer())
                            if "academic_pdfs" not in data:
                                data["academic_pdfs"] = []
                            data["academic_pdfs"].append({
                                "subject": subject,
                                "semester": semester,
                                "description": description,
                                "file_path": file_path,
                                "upload_date": datetime.now().strftime("%Y-%m-%d")
                            })
                            save_data(data)
                            st.success(f"PDF uploaded successfully for {subject}!")
                            st.rerun()
                    else:
                        st.error("Please select a PDF file to upload")
        st.markdown("<div class='stSubheader'><h3>📚 Available Study Materials</h3></div>", unsafe_allow_html=True)
        if "academic_pdfs" in data and data["academic_pdfs"]:
            for i, pdf in enumerate(list(data["academic_pdfs"])):
                if os.path.exists(pdf["file_path"]):
                    col1, col2 = st.columns([0.8, 0.2])
                    with col1:
                        st.markdown(f"""
                        <div class='card resource-card'>
                            <h3>{pdf['subject']} ({pdf['semester']} Semester)</h3>
                            <p>📅 Uploaded: {pdf['upload_date']}</p>
                            <p>{pdf['description']}</p>
                        </div>
                        """, unsafe_allow_html=True)
                    with col2:
                        st.download_button(
                            label="Download PDF",
                            data=open(pdf["file_path"], "rb").read(),
                            file_name=os.path.basename(pdf["file_path"]),
                            mime="application/pdf",
                            key=f"download_pdf_{i}"
                        )
                        if st.button("Delete PDF", key=f"delete_pdf_{i}"):
                            try:
                                os.remove(pdf["file_path"])
                                data["academic_pdfs"].pop(i)
                                save_data(data)
                                st.success("PDF deleted successfully!")
                                st.rerun()
                            except Exception as e:
                                st.error(f"Error deleting PDF: {e}")
                else:
                    st.warning(f"File not found: {pdf['subject']}")
        else:
            st.info("No study materials available yet.")
    else:
        st.markdown("<div class='stSubheader'><h3>📚 Available Study Materials</h3></div>", unsafe_allow_html=True)
        if "academic_pdfs" in data and data["academic_pdfs"]:
            for pdf in data["academic_pdfs"]:
                if os.path.exists(pdf["file_path"]):
                    st.markdown(f"""
                    <div class='card resource-card'>
                        <h3>{pdf['subject']} ({pdf['semester']} Semester)</h3>
                        <p>📅 Uploaded: {pdf['upload_date']}</p>
                        <p>{pdf['description']}</p>
                    </div>
                    """, unsafe_allow_html=True)
                    st.download_button(
                        label="Download PDF",
                        data=open(pdf["file_path"], "rb").read(),
                        file_name=os.path.basename(pdf["file_path"]),
                        mime="application/pdf",
                        key=f"download_pdf_public_{pdf['file_path']}"
                    )
                else:
                    st.warning(f"File not found: {pdf['subject']}")
        else:
            st.info("No study materials available yet.")
        st.warning("Please login as admin to upload study materials")

def show_transport(data):
    st.markdown("<div class='stSubheader'><h2>🚌 Campus Transport</h2></div>", unsafe_allow_html=True)
    if st.session_state.get('authenticated', False):
        with st.expander("📤 Upload Updated Bus Routes (Admin Only)"):
            with st.form("transport_pdf_upload_form"):
                pdf_file = st.file_uploader("Choose a PDF file with updated routes", type="pdf")
                title = st.text_input("Document Title (e.g., '2025 Bus Routes Update')")
                description = st.text_area("Description of changes")
                if st.form_submit_button("Upload PDF"):
                    if pdf_file is not None:
                        os.makedirs("transport_docs", exist_ok=True)
                        file_path = os.path.join("transport_docs", f"{title.replace(' ', '_')}.pdf")
                        with open(file_path, "wb") as f:
                            f.write(pdf_file.getbuffer())
                        if "transport_pdfs" not in data:
                            data["transport_pdfs"] = []
                        data["transport_pdfs"].append({
                            "title": title,
                            "description": description,
                            "file_path": file_path,
                            "upload_date": datetime.now().strftime("%Y-%m-%d")
                        })
                        save_data(data)
                        st.success("Bus routes PDF uploaded successfully!")
                        st.rerun()
                    else:
                        st.error("Please select a PDF file to upload")
    col1, col2 = st.columns(2)
    with col1:
        st.markdown("<div class='stSubheader'><h3>🚏 Bus Routes</h3></div>", unsafe_allow_html=True)
        for route in data["transport"]["bus_routes"]:
            st.markdown(f"""
            <div class='card'>
                <h3>{route[0]}</h3>
                <p>📍 <strong>Route:</strong> {route[1]}</p>
                <p>🕒 <strong>Timings:</strong> {route[2]}</p>
            </div>
            """, unsafe_allow_html=True)
    with col2:
        st.markdown("<div class='stSubheader'><h3>🚐 Shuttle Services</h3></div>", unsafe_allow_html=True)
        for shuttle in data["transport"]["shuttle_times"]:
            st.markdown(f"""
            <div class='card'>
                <h3>{shuttle[0]}</h3>
                <p>⏰ <strong>Operating Hours:</strong> {shuttle[1]}</p>
                <p>🔄 <strong>Frequency:</strong> {shuttle[2]}</p>
            </div>
            """, unsafe_allow_html=True)
    st.markdown("<div class='stSubheader'><h3>📄 Updated Bus Route Documents</h3></div>", unsafe_allow_html=True)
    if "transport_pdfs" in data and data["transport_pdfs"]:
        for i, pdf in enumerate(list(data["transport_pdfs"])):
            if os.path.exists(pdf["file_path"]):
                col1, col2 = st.columns([0.8, 0.2])
                with col1:
                    st.markdown(f"""
                    <div class='card resource-card'>
                        <h3>{pdf['title']}</h3>
                        <p>📅 Uploaded: {pdf['upload_date']}</p>
                        <p>{pdf['description']}</p>
                    </div>
                    """, unsafe_allow_html=True)
                with col2:
                    st.download_button(
                        label="Download PDF",
                        data=open(pdf["file_path"], "rb").read(),
                        file_name=os.path.basename(pdf["file_path"]),
                        mime="application/pdf",
                        key=f"download_transport_pdf_{i}"
                    )
                    if st.session_state.get('authenticated', False):
                        if st.button("Delete PDF", key=f"delete_transport_pdf_{i}"):
                            try:
                                os.remove(pdf["file_path"])
                                data["transport_pdfs"].pop(i)
                                save_data(data)
                                st.success("PDF deleted successfully!")
                                st.rerun()
                            except Exception as e:
                                st.error(f"Error deleting PDF: {e}")
            else:
                st.warning(f"File not found: {pdf['title']}")
    else:
        st.info("No updated bus route documents available yet.")

def show_food_stay(data):
    st.markdown("<div class='stSubheader'><h2>🍔 Food & Stay</h2></div>", unsafe_allow_html=True)
    tab1, tab2 = st.tabs(["Food Options", "Hostel Information"])
    with tab1:
        st.markdown("<div class='stSubheader'><h3>🍽️ Campus Canteens</h3></div>", unsafe_allow_html=True)
        for canteen in data["food"]["canteens"]:
            st.markdown(f"""
            <div class='card'>
                <h3>{canteen[0]}</h3>
                <p>🕒 <strong>Timings:</strong> {canteen[1]}</p>
                <p>📍 <strong>Location:</strong> {canteen[2]}</p>
                <p>🍴 <strong>Menu:</strong> {canteen[3]}</p>
            </div>
            """, unsafe_allow_html=True)   
    with tab2:
        st.markdown("<div class='stSubheader'><h3>🏠 Hostel Facilities</h3></div>", unsafe_allow_html=True)
        for hostel in data["food"]["hostels"]:
            st.markdown(f"""
            <div class='card'>
                <h3>{hostel[0]}</h3>
                <p>🛏️ <strong>Capacity:</strong> {hostel[1]} students</p>
                <p>👥 <strong>Room Type:</strong> {hostel[2]}</p>
                <p>❄️ <strong>Facilities:</strong> {hostel[3]}</p>
            </div>
            """, unsafe_allow_html=True)

def show_lost_found(data):
    st.markdown("<div class='stSubheader'><h2>🔍 Lost & Found</h2></div>", unsafe_allow_html=True)
    with st.expander("➕ Report Lost Item"):
        with st.form("lost_item_form"):
            item_name = st.text_input("Item Name")
            description = st.text_area("Description")
            location = st.text_input("Where did you lose it?")
            date_lost = st.date_input("Date Lost")
            contact = st.text_input("Your Contact Information")
            if st.form_submit_button("Submit Report"):
                new_item = [item_name, location, str(date_lost), "Not claimed"]
                data["lost_found"].append(new_item)
                save_data(data)
                st.success("Lost item reported successfully!")
    st.markdown("<div class='stSubheader'><h3>📝 Lost Items Log</h3></div>", unsafe_allow_html=True)
    for item in data["lost_found"]:
        if len(item) >= 4:
            status_color = "completed" if item[3] == "Claimed" else "ongoing"
            st.markdown(f"""
            <div class='card'>
                <div style="display: flex; justify-content: space-between; align-items: center;">
                    <h3>{item[0]}</h3>
                    <span class='status-badge {status_color}'>{item[3]}</span>
                </div>
                <p>📍 <strong>Location:</strong> {item[1]}</p>
                <p>🗓️ <strong>Date:</strong> {item[2]}</p>
            </div>
            """, unsafe_allow_html=True)

def show_emergency(data):
    st.markdown("<div class='stSubheader'><h2>🚨 Emergency Services</h2></div>", unsafe_allow_html=True)
    st.markdown("<div class='stSubheader'><h3>📞 Emergency Contacts</h3></div>", unsafe_allow_html=True)
    for contact in data["emergency"]["data"]:
        if len(contact) >= 4:
            st.markdown(f"""
            <div class='card emergency-card'>
                <h3>{contact[0]}</h3>
                <p>📞 <strong>Number:</strong> <span style="font-size: 1.2rem; font-weight: bold;">{contact[1]}</span></p>
                <p>⏰ <strong>Available:</strong> {contact[2]}</p>
                <p>📍 <strong>Location:</strong> {contact[3]}</p>
            </div>
            """, unsafe_allow_html=True)

def main():
    data = load_data()
    show_header()
    current_page = show_sidebar()
    is_admin = authenticate()
    if is_admin:
        show_admin_panel(data)
    if current_page == "🏠 Dashboard":
        show_dashboard(data)
    elif current_page == "📞 Directory":
        show_directory(data)
    elif current_page == "📅 Events":
        show_events(data)
    elif current_page == "📚 Academics":
        show_academics(data)
    elif current_page == "🚌 Transport":
        show_transport(data)
    elif current_page == "🍔 Food & Stay":
        show_food_stay(data)
    elif current_page == "🚨 Emergency":
        show_emergency(data)
    elif current_page == "🔍 Lost & Found":
        show_lost_found(data)
    st.markdown("""
    <hr style="margin: 2rem 0; border: none; border-top: 1px solid #444;">
    <div style="text-align: center; color: #888; font-size: 0.9rem;">
        <p>Vignan Institute of Technology and Science • © 2025 • v1.3.0</p>
        <p>For technical issues, contact <a href="mailto:sricharankarthikk@gmail.com">IT Helpdesk</a></p>
    </div>
    """, unsafe_allow_html=True)

if __name__ == "__main__":
    main()
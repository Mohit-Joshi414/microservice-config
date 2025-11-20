Title:

Logistics Route Optimization Explanation Agent

Problem Statement:

Logistics companies optimize delivery routes using algorithms but often lack clear explanations for route decisions, complicating operational planning and exception handling. An Al agent that generates human-readable rationales explaining route choices and suggesting improvements will increase transparency and trust.

Data Requirements:

Route optimization outputs, delivery constraints, and related logistics data.

(Feel free to generate synthetic data as well from the provided access)

Expected Output:

Detailed textual explanations for each route segment, including reasoning, alternatives considered, and improvement suggestions formatted for logistics teams.

Be creative to go beyond as you solve the above needs

import streamlit as st
import json
import requests
from dotenv import load_dotenv
load_dotenv()

st.set_page_config(page_title='Logistics Explanation Agent', layout='wide')

st.title('Logistics Route Explanation — Streamlit Frontend')
st.markdown('Paste a route JSON (or use the sample) and click **Explain** to call the backend `/explain` API.')

sample = {
    "route_id": "R-001",
    "vehicle": {"id": "V1", "capacity_kg": 12000},
    "driver": {"id": "D1", "hours_available": 9},
    "stops": [
        {"stop_id": "Depot"},
        {"stop_id": "C1", "tw": ["09:00", "11:00"], "demand_kg": 1200},
        {"stop_id": "C2", "tw": ["08:00", "10:00"], "demand_kg": 800},
        {"stop_id": "C3", "tw": ["13:00", "15:00"], "demand_kg": 3000}
    ],
    "legs": [
        {"from": "Depot", "to": "C2", "distance_km": 45, "travel_time_min": 70, "expected_arrival": "2025-11-20T08:20:00+05:30"},
        {"from": "C2", "to": "C1", "distance_km": 10, "travel_time_min": 25, "expected_arrival": "2025-11-20T09:05:00+05:30"},
        {"from": "C1", "to": "C3", "distance_km": 150, "travel_time_min": 150, "expected_arrival": "2025-11-20T13:45:00+05:30"}
    ],
    "optimizer_metrics": {"total_distance_km": 205, "eta_finish": "2025-11-20T16:30:00+05:30", "estimated_cost": 420}
}

input_col, preview_col = st.columns([2, 1])

with input_col:
    text = st.text_area('Route JSON', value=json.dumps(sample, indent=2), height=360)
    endpoint = st.text_input('Backend endpoint', value='http://localhost:5000/explain')
    explain_btn = st.button('Explain')

with preview_col:
    st.subheader('Quick preview')
    try:
        preview = json.loads(text)
        st.json(preview)
    except Exception:
        st.info('Invalid JSON')

if explain_btn:
    try:
        payload = json.loads(text)
    except Exception as e:
        st.error(f'Invalid JSON: {e}')
    else:
        with st.spinner('Calling backend...'):
            try:
                resp = requests.post(endpoint, json=payload, timeout=20)
                resp.raise_for_status()
                data = resp.json()
            except Exception as e:
                st.error(f'Error calling backend: {e}')
                data = None

        if data:
            st.success('Explanation received')
            st.subheader(f"Route: {data.get('route_id')}")
            st.write(data.get('summary'))

            for idx, seg in enumerate(data.get('segments', [])):
                with st.expander(f"Leg {idx+1}: {seg.get('leg', {}).get('from')} → {seg.get('leg', {}).get('to')}"):
                    leg = seg.get('leg', {})
                    feats = seg.get('features', {})
                    st.markdown('**Narrative**')
                    st.write(seg.get('narrative'))
                    st.markdown('**Features**')
                    st.write({
                        'distance_km': feats.get('distance_km'),
                        'travel_time_min': feats.get('travel_time_min'),
                        'eta': leg.get('expected_arrival'),
                        'time_window_slack_min': feats.get('time_window_slack_min'),
                        'capacity_left_kg': feats.get('capacity_left_kg'),
                        'cost_estimate': feats.get('cost_estimate')
                    })
                    st.markdown('**Suggestions**')
                    suggestions = seg.get('suggestions', [])
                    if suggestions:
                        for s in suggestions:
                            st.write('- ' + s)
                    else:
                        st.write('No suggestions')

                    st.markdown('**Alternatives (preview)**')
                    alts = seg.get('alternatives', [])
                    if alts:
                        for a in alts:
                            st.write('- ' + a.get('type', 'alt') + (f" (swap_index={a.get('swap_index')})" if a.get('swap_index') is not None else ''))
                    else:
                        st.write('No alternatives')

            st.markdown('---')
            st.json(data)

if st.button('Show sample route in console'):
    st.write(json.dumps(sample, indent=2))

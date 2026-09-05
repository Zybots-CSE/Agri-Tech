# AgriShield – Smart Climate Resilient Agricultural Early Warning System

Welcome to the **AgriShield** project repository. AgriShield is an IoT powered flood prediction and automated early warning platform designed specifically to protect paddy farmers and rural agricultural communities from climate unpredictability and extreme weather events.

---

## What is AgriShield?

Extreme weather events, such as flash floods and heavy monsoonal rains, inflict massive damage on Sri Lanka's agricultural sector. When paddy fields flood without warning, farmers lose crops, machinery, and months of hard work. 

**AgriShield solves this problem using a 3-part system:**
1. **Solar Powered Sensor Nodes:** Installed directly in paddy fields and water channels to monitor water levels and soil moisture in real time.
2. **Cloud AI Risk Prediction:** Combines live field measurements, local weather forecasts, and upstream reservoir discharge data to predict flood risks up to 6 hours in advance.
3. **Simple Language SMS Warnings:** Sends clear, direct warnings and practical instructions in **Sinhala and Tamil** to farmers' basic feature phones—no smartphone or internet connection required.

---

## Proposal & Presentation Assets

* **File Name for Final Proposal Submission:** 
* **Platform Category:** Mechanical + IoT Platform.

### Core Presentation Structure (10 Slides)

* **Slide 1: Title** – AgriShield: Smart Climate-Resilient Agricultural Early Warning System.
* **Slide 2: Background & Problem Identification** – Highlights recent extreme weather impacts (e.g., severe Maha season flooding), the lack of field-level early warnings, and why app-based solutions fail in rural areas.
* **Slide 3: Proposed Innovative Solution (Part 1)** – End-to-end flow: Solar ESP32 nodes -> Cloud AI Engine -> Sinhala/Tamil SMS alerts.
* **Slide 4: Proposed Innovative Solution (Part 2)** – Enclosure hardware design and the Officer Web GIS Dashboard.
* **Slide 5: Implementation Plan** – 4-phase roadmap covering prototyping, cloud development, pilot testing, and nationwide scale-up.
* **Slide 6: Market Potential / Business Model (Part 1)** – Target audience (1.5M+ paddy farmers, Agrarian Services, AgTech insurers) and go-to-market plan.
* **Slide 7: Market Potential / Business Model (Part 2)** – Revenue model via B2G grants, insurance API access, and maintenance contracts.
* **Slide 8: Social, Economic & Environmental Impact** – Multi-lingual accessibility, protecting rural livelihoods, and optimizing irrigation water management.
* **Slide 9: Technical Feasibility** – Proven hardware stack (ESP32 + SIM800L + JSN-SR04T) with solar autonomy and serverless cloud APIs.
* **Slide 10: Expected Outcomes** – 2 to 6-hour advance flood warnings, 100% reach on feature phones, and low-cost unit deployment (< Rs. 15,000 per node).

---

### How Farmers Receive Predictions and Recommendations
Over 75% of smallholder paddy farmers rely on standard feature phones. AgriShield sends immediate, actionable text messages in native languages.

#### Sinhala SMS Example
> **අවධානයයි! (AgriShield - POL-112)**  
> ඉදිරි පැය 3 ඇතුලත ඔබගේ කුඹුරට ජලය පිරීමේ අධික අවදානමක් ඇත.  
> **නිර්දේශිත පියවර:**  
> 1. කුඹුරේ නියරවල්වල ජල නාලිකා වහාම විවෘත කරන්න.  
> 2. පොහොර යෙදීම අත්හිටුවන්න.  
> 3. අස්වැන්න නෙලූ ධාන්‍ය උස් බිම් වෙත ගෙනයන්න.

#### Tamil SMS Example
> **எச்சரிக்கை! (AgriShield - POL-112)**  
> அடுத்த 3 மணிநேரத்தில் உங்கள் வயலில் வெள்ளம் சூழும் அபாயம் உள்ளது.  
> **பரிந்துரைக்கப்பட்ட நடவடிக்கைகள்:**  
> 1. வயல் வடிகால்களை உடனடியாக திறக்கவும்.  
> 2. உரம் போடுவதை தவிர்க்கவும்.  
> 3. அறுவடை செய்த நெல்லை பாதுகாப்பான இடத்திற்கு மாற்றவும்.

#### Automatic Action Matrix

| Field Status | What System Detects | What Action the Farmer Receives |
| :--- | :--- | :--- |
| **Rising Water Level** | Water rising rapidly over 1 hour | Open edge drainage gates; hold off on applying fertilizer/pesticides. |
| **Severe Flood Risk** | Predicted inundation >50cm in 2 hours | Move harvested crops and machinery to high ground immediately. |
| **Dry Spell / Low Moisture**| Soil moisture drops below 20% | Follow water-saving irrigation; apply light watering rather than flooding. |

---

## Software & Machine Learning Architecture

For a technical lead or software engineer, the backend processes data using the following pipeline:

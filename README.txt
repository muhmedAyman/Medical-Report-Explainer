MEDICAL REPORT EXPLAINER
========================

A Streamlit app that reads a lab report and explains the results in simple English or Arabic. It finds the abnormal values, explains what they may mean, looks for patterns between tests, and shows hospitals and clinics near the user.

Techniques used: RAG (FAISS), LangChain chains, output parsers, and a 4-bit quantized LLM.


WHAT IT DOES
------------

- Takes the report as pasted text or an uploaded PDF (text-based PDFs only).
- Reads the tests with regex rules. The LLM is only a backup when the text is messy.
- Compares each value with its reference range in plain Python. The model never decides if a value is high or low.
- Has a knowledge base of 48 tests in 9 groups: Complete Blood Count, Blood Sugar, Lipid Profile, Liver Function, Kidney Function, Thyroid, Electrolytes and Minerals, Vitamins and Iron, Inflammation and Clotting.
- Explains results in English or Arabic. The page switches to right-to-left for Arabic.
- Uses a different range for male and female. If the report prints its own range, that one is used first.
- Shows an alert for critical values.
- Finds patterns between several results with fixed rules (for example low hemoglobin + low MCV + low ferritin).
- Gives questions the patient can ask the doctor.
- Suggests which type of doctor to visit and lists nearby hospitals, clinics and doctors with Google Maps links.
- Lets the user download the result as a Markdown file.


HOW IT WORKS
------------

1. The user pastes the report or uploads a PDF (read with pypdf), then picks sex, age, language and extraction mode.
2. Regex rules extract the test name, value, unit and range. In Auto mode, if fewer than 3 tests are found, the LLM reads the report and returns JSON. Its output is checked against the original text so it can't invent tests.
3. Each test name is matched to the knowledge base using aliases (for example Hb, Hgb, Haemoglobin). Units are normalized.
4. Python marks each value as Low, High or Normal and calculates how far it is from the range in percent. Critical values are marked urgent.
5. Abnormal results are sorted (critical first, then the furthest from the range). The first 10 are explained.
6. For each abnormal test, FAISS finds the most related entries in the knowledge base. The LLM writes the explanation from them in fixed sections: Meaning, Your result, Common reasons, Related results, Ask your doctor.
7. Fixed if-rules look for combinations of results. The text describes the pattern and does not diagnose.
8. The LLM writes a short overall summary. If the model fails, a fallback summary is used.
9. The app shows nearby places (see below).
10. The result can be downloaded as Markdown.


NEARBY HOSPITALS AND CLINICS
----------------------------

No Google API is used, so no API key is needed. The app uses two free OpenStreetMap services:

- Nominatim turns a place name (like "Dokki, Giza") into coordinates. The user can also type coordinates.
- Overpass API returns hospitals, clinics and doctors around that point. Default radius is 5 km (up to 30 km). If fewer than 5 places are found, the radius is doubled.

The distance to each place is calculated with the Haversine formula and the list is sorted from the nearest. The specialty tag is matched with the doctor type suggested for the abnormal tests. Google Maps is only used for the links (search and directions).

The Overpass servers are public and sometimes busy, so the code tries several mirrors and repeats the round once. Only the location is sent to these services, never the report. OpenStreetMap data can be incomplete in some areas.


TECH STACK
----------

- LLM: mistralai/Mistral-Nemo-Instruct-2407, 4-bit (NF4) with bitsandbytes. Can be changed with the MODEL_ID environment variable.
- Embeddings: sentence-transformers/all-MiniLM-L6-v2 (CPU)
- Vector store: FAISS
- Framework: LangChain (LLMChain, PromptTemplate, StructuredOutputParser)
- UI: Streamlit, shared with ngrok
- PDF reading: pypdf
- Places: OpenStreetMap (Nominatim and Overpass) with requests


PROJECT STRUCTURE
-----------------

    medical_explainer/
    ├── medical_core.py       # extraction rules, units, range comparison, patterns
    ├── engine.py             # model, embeddings, FAISS index, chains
    ├── locator.py            # nearby hospitals, clinics and doctors
    ├── app.py                # Streamlit app
    └── knowledge_base.json   # 48 tests: ranges, meanings, reasons, related tests (English and Arabic)



HOW TO RUN (KAGGLE)
-------------------

1. Create a Kaggle notebook and upload Medical_Report_Explainer.ipynb.
2. In Settings choose Accelerator = GPU T4 x2 and Internet = On.
3. Add your ngrok token as a Kaggle secret named NGROK_TOKEN.
4. Run the cells from top to bottom. The test cells check the core logic and the locator without loading the model.
5. Open the public link printed by the last cell. The first visit loads the model, which takes a few minutes.


USING THE APP
-------------

1. Choose sex, age, language and extraction mode in the sidebar.
2. Paste the report text or upload a PDF.
3. Press the analyze button and read the results, patterns, summary and questions for the doctor.
4. Type your city or area to see nearby places.
5. Download the result if you want to keep it.


LIMITATIONS
-----------

- The app explains results in simple language. It does not diagnose, recommend treatment, or replace a doctor.
- The knowledge base covers 48 common tests. Other tests are shown but not explained.
- Reference ranges differ between labs. The ones in the knowledge base are general adult ranges, which is why the range printed in the report is preferred. They are not for children (the app shows a warning when age is under 18).
- Scanned PDFs (images without text) are not supported.
- The model needs a GPU, so the app is meant for Kaggle or Colab.
- Nearby places depend on OpenStreetMap data and may be incomplete or out of date.

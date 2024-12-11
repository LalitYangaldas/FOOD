import streamlit as st
from PIL import Image
import pytesseract
import re
import openai

# OpenAI API Key
openai.api_key = "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"
# Tesseract Path
pytesseract.pytesseract.tesseract_cmd = r"C:/Program Files/Tesseract-OCR/tesseract.exe"

# Helper Functions
def extract_text_from_image(image):
    """Extract text from an uploaded image using OCR."""
    try:
        text = pytesseract.image_to_string(image)
        return text.strip()
    except Exception as e:
        return f"Error extracting text: {str(e)}"

def classify_ingredients(ingredients):
    """Classify ingredients into high and low risk based on common terms."""
    high_risk = []
    low_risk = []
    for ingredient in ingredients:
        if re.search(r'sugar|sodium|salt|high fructose corn syrup', ingredient, re.IGNORECASE):
            high_risk.append(ingredient)
        else:
            low_risk.append(ingredient)
    return high_risk, low_risk

def decode_scientific_terms(ingredients):
    """Decode scientific ingredient names using OpenAI."""
    decoded_ingredients = {}
    for ingredient in ingredients:
        try:
            response = openai.Completion.create(
                engine="text-davinci-003",
                prompt=f"Explain what {ingredient} is in simple terms.",
                max_tokens=50,
                temperature=0.5
            )
            decoded_ingredients[ingredient] = response.choices[0].text.strip()
        except Exception as e:
            decoded_ingredients[ingredient] = f"Error decoding: {str(e)}"
    return decoded_ingredients

def suggest_consumption(high_risk):
    """Provide consumption recommendations based on high-risk ingredients."""
    if high_risk:
        return "Limit consumption of these high-risk ingredients: " + ", ".join(high_risk)
    return "This food appears to be generally healthy."

# Streamlit Web App
st.set_page_config(page_title="Nutritional Label Analyzer", layout="centered")
st.title("Nutritional Label Analyzer")
st.write("Upload an image of a food nutritional label or enter text to analyze its health impact.")

# Input Section
input_method = st.radio("Choose your input method:", ("Upload Image", "Enter Text"))

if input_method == "Upload Image":
    uploaded_image = st.file_uploader("Upload an image of the nutritional label:", type=["jpg", "png", "jpeg"])
    if uploaded_image is not None:
        image = Image.open(uploaded_image)
        st.image(image, caption="Uploaded Image", use_column_width=True)
        with st.spinner("Extracting text..."):
            extracted_text = extract_text_from_image(image)
        st.text_area("Extracted Text:", value=extracted_text, height=200)
else:
    extracted_text = st.text_area("Enter nutritional label text below:", height=200)

# Analysis Button
if st.button("Analyze"):
    if extracted_text:
        # Extract and process ingredients
        with st.spinner("Classifying ingredients..."):
            ingredients = re.findall(r'\b[A-Za-z][A-Za-z0-9\s\-]*\b', extracted_text)
            high_risk, low_risk = classify_ingredients(ingredients)
        
        with st.spinner("Decoding scientific terms..."):
            decoded_terms = decode_scientific_terms(ingredients)

        # Display Results
        st.subheader("Ingredient Classification")
        st.write(f"High Risk: {', '.join(high_risk) if high_risk else 'None'}")
        st.write(f"Low Risk: {', '.join(low_risk) if low_risk else 'None'}")

        st.subheader("Decoded Ingredients")
        for ingredient, decoded in decoded_terms.items():
            st.write(f"{ingredient}: {decoded}")

        st.subheader("Recommendations")
        recommendations = suggest_consumption(high_risk)
        st.write(recommendations)
    else:
        st.error("Please provide text or upload a valid image.")

st.write("💡 Note: This app analyzes nutritional information for educational purposes only.")


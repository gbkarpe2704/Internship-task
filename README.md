# Internship-task

Sure! Here's a step-by-step guide on how to build a Django REST framework (DRF) API integrated with MongoDB, capable of accepting PDF files, extracting nouns and verbs, saving data to the database, and deploying it on a platform with GitHub Actions for continuous deployment. We'll use libraries like `pdfplumber` for PDF extraction and `spaCy` for natural language processing (to identify nouns and verbs).

### 1. **Set up the Django Project**
First, let's set up the Django project and install necessary dependencies.

1. Create a new Django project:

```bash
django-admin startproject pdf_extractor
cd pdf_extractor
```

2. Create a new app for your API:

```bash
python manage.py startapp api
```

3. Install the required dependencies:

```bash
pip install django djangorestframework mongoengine pdfplumber spacy
```

4. Install `spaCy` and download the necessary model for natural language processing:

```bash
python -m spacy download en_core_web_sm
```

5. Set up MongoDB integration with Django using `mongoengine`. In your settings file (`settings.py`), configure the database connection:

```python
DATABASES = {
    'default': {
        'ENGINE': 'djongo',
        'NAME': 'pdf_extractor_db',
        'ENFORCE_SCHEMA': False,
    }
}
```

---

### 2. **Model and API for PDF Data**

Now, create the model for storing the email and extracted nouns and verbs. In `api/models.py`:

```python
from mongoengine import Document, StringField, ListField
import spacy
import pdfplumber

# Load spaCy model
nlp = spacy.load('en_core_web_sm')

class ExtractedData(Document):
    email = StringField(required=True, unique=True)
    nouns = ListField(StringField())
    verbs = ListField(StringField())

    def extract_text_from_pdf(self, pdf_file):
        with pdfplumber.open(pdf_file) as pdf:
            text = ""
            for page in pdf.pages:
                text += page.extract_text()
            return text

    def extract_nouns_verbs(self, text):
        doc = nlp(text)
        nouns = [token.text for token in doc if token.pos_ == "NOUN"]
        verbs = [token.text for token in doc if token.pos_ == "VERB"]
        return nouns, verbs
```

---

### 3. **Serializers for the API**

Create serializers to handle PDF files and store the results. In `api/serializers.py`:

```python
from rest_framework import serializers
from .models import ExtractedData

class ExtractedDataSerializer(serializers.ModelSerializer):
    class Meta:
        model = ExtractedData
        fields = ['email', 'nouns', 'verbs']

class PDFUploadSerializer(serializers.Serializer):
    email = serializers.EmailField()
    pdf_file = serializers.FileField()
```

---

### 4. **Views for Handling File Upload and Text Extraction**

Now, create the API views in `api/views.py`:

```python
from rest_framework import status, views
from rest_framework.response import Response
from rest_framework.parsers import MultiPartParser, FormParser
from .models import ExtractedData
from .serializers import ExtractedDataSerializer, PDFUploadSerializer
from django.core.exceptions import ValidationError

class PDFUploadView(views.APIView):
    parser_classes = (MultiPartParser, FormParser)

    def post(self, request, *args, **kwargs):
        serializer = PDFUploadSerializer(data=request.data)
        if serializer.is_valid():
            email = serializer.validated_data['email']
            pdf_file = serializer.validated_data['pdf_file']

            # Extract text from the PDF
            extracted_data = ExtractedData()
            text = extracted_data.extract_text_from_pdf(pdf_file)

            # Extract nouns and verbs
            nouns, verbs = extracted_data.extract_nouns_verbs(text)

            # Save to database
            try:
                # Check if the email already exists
                existing_entry = ExtractedData.objects(email=email).first()
                if existing_entry:
                    return Response({"message": "Email already exists"}, status=status.HTTP_400_BAD_REQUEST)
                
                extracted_data.email = email
                extracted_data.nouns = nouns
                extracted_data.verbs = verbs
                extracted_data.save()

                # Return the serialized data
                return Response(ExtractedDataSerializer(extracted_data).data, status=status.HTTP_201_CREATED)

            except Exception as e:
                return Response({"error": str(e)}, status=status.HTTP_400_BAD_REQUEST)
        else:
            return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

---

### 5. **URLs**

In `api/urls.py`, create the endpoint for uploading PDFs:

```python
from django.urls import path
from .views import PDFUploadView

urlpatterns = [
    path('upload/', PDFUploadView.as_view(), name='pdf-upload'),
]
```

In `pdf_extractor/urls.py`, include the `api` URLs:

```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]
```

---

### 6. **Frontend for Testing**

Create a simple frontend using HTML and JavaScript to upload a PDF file and display the extracted nouns and verbs. Create a file named `index.html`:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>PDF Text Extractor</title>
</head>
<body>
    <h1>Upload PDF to Extract Nouns and Verbs</h1>
    <form id="uploadForm">
        <input type="email" id="email" required placeholder="Enter your email" />
        <input type="file" id="pdfFile" accept=".pdf" required />
        <button type="submit">Upload</button>
    </form>
    <h2>Extracted Data</h2>
    <p><strong>Nouns:</strong> <span id="nouns"></span></p>
    <p><strong>Verbs:</strong> <span id="verbs"></span></p>

    <script>
        document.getElementById('uploadForm').addEventListener('submit', function(event) {
            event.preventDefault();
            const email = document.getElementById('email').value;
            const pdfFile = document.getElementById('pdfFile').files[0];

            const formData = new FormData();
            formData.append('email', email);
            formData.append('pdf_file', pdfFile);

            fetch('http://127.0.0.1:8000/api/upload/', {
                method: 'POST',
                body: formData
            })
            .then(response => response.json())
            .then(data => {
                document.getElementById('nouns').textContent = data.nouns.join(', ');
                document.getElementById('verbs').textContent = data.verbs.join(', ');
            })
            .catch(error => console.error('Error:', error));
        });
    </script>
</body>
</html>
```

---

### 7. **Deployment with GitHub Actions**

1. **Create a `Dockerfile`** to containerize the app:

```dockerfile
FROM python:3.8-slim

WORKDIR /app
COPY . /app/

RUN pip install --no-cache-dir -r requirements.txt

EXPOSE 8000

CMD ["python", "manage.py", "runserver", "0.0.0.0:8000"]
```

2. **Create a `docker-compose.yml`** for MongoDB and the app:

```yaml
version: '3'

services:
  db:
    image: mongo:latest
    container_name: mongo
    ports:
      - "27017:27017"
    networks:
      - app-network

  web:
    build: .
    container_name: django-api
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
    networks:
      - app-network

networks:
  app-network:
    driver: bridge
```

3. **Create GitHub Actions Workflow (`.github/workflows/deploy.yml`)**:

```yaml
name: Deploy Django App

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
        with:
          python-version: '3.8'
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
      - name: Run migrations
        run: |
          python manage.py migrate
      - name: Deploy app
        run: |
          docker-compose up --build -d
```

4. Push your changes to GitHub.

---

### 8. **Deploy on Heroku**

1. Set up a Heroku app.
2. Push your app to Heroku with Git:
   
```bash
git remote add heroku https://git.heroku.com/your-app-name.git
git push heroku main
```

Heroku should automatically deploy the app, and you can visit the URL it provides.

---

### 9. **Test**

Now you should be able to:

1. Upload a PDF through the frontend.
2. See the extracted nouns and verbs displayed on the page.

You can access your Heroku app using the provided URL.

---

### Conclusion

This process outlines how to set up a Django app integrated with MongoDB, extract content from PDFs, store that data, and deploy it with GitHub Actions. The deployment process with Docker and Heroku ensures your app is easily accessible online.

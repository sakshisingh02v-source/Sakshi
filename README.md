# Chemical Equipment Visualizer

This project is a web and desktop application for visualizing chemical equipment data from CSV files. It consists of a Django backend with REST API, a React web frontend, and a PyQt desktop frontend.

## Architecture

```
chemical-equipment-visualizer/
│
├── backend/manage.py                # Django backend and analytics API
│   ├── requirements.txt
│   ├── backend/
│   │   ├── __init__.py
│   │   ├── settings.py
│   │   ├── urls.py
│   │   └── wsgi.py
│   └── analytics/
│       ├── __init__.py
│       ├── models.py
│       ├── views.py
│       ├── serializers.py
│       ├── urls.py
│       └── utils.py
│
├── web-frontend/            # React web app
│   ├── package.json
│   └── src/
│       ├── App.js
│       ├── api.js
│       └── components/
│           ├── Upload.js
│           ├── Summary.js
│           └── Charts.js
│
├── desktop-frontend/        # PyQt desktop app
│   ├── main.py
│   └── requirements.txt
│
├── sample_equipment_data.csv
│
└── README.md
```

---

## Backend (Django/DRF)

- REST API for file uploads and analytics history.
- On CSV upload: parses file, computes summary statistics, stores up to 5 recent analyses.

Install requirements:
```
cd backend
pip install -r requirements.txt
```

**Key dependencies:**  
- django  
- djangorestframework  
- pandas  
- reportlab

### `analytics/models.py`

```python
from django.db import models

class Dataset(models.Model):
    filename = models.CharField(max_length=255)
    analysis_result = models.JSONField()
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        ordering = ["-created_at"]

    def __str__(self):
        return self.filename

```

### `analytics/utils.py`

```python
import pandas as pd

REQUIRED_COLUMNS = {"Flowrate", "Pressure", "Temperature", "Type"}

def analyze_csv(file_obj):
    df = pd.read_csv(file_obj)

    if not REQUIRED_COLUMNS.issubset(df.columns):
        missing = REQUIRED_COLUMNS - set(df.columns)
        raise ValueError(f"Missing columns: {', '.join(missing)}")

    return {
        "equipment_count": int(len(df)),
        "average_flowrate": float(df["Flowrate"].mean()),
        "average_pressure": float(df["Pressure"].mean()),
        "average_temperature": float(df["Temperature"].mean()),
        "equipment_types": df["Type"].value_counts().to_dict(),
    }

```

### `analytics/views.py`

```python
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework.permissions import IsAuthenticated
from rest_framework import status

from .models import Dataset
from .utils import analyze_csv

class UploadCSVView(APIView):
    permission_classes = [IsAuthenticated]

    def post(self, request):
        if "file" not in request.FILES:
            return Response(
                {"error": "CSV file not provided"},
                status=status.HTTP_400_BAD_REQUEST
            )

        csv_file = request.FILES["file"]

        try:
            summary = analyze_csv(csv_file)
        except Exception as exc:
            return Response(
                {"error": str(exc)},
                status=status.HTTP_400_BAD_REQUEST
            )

        Dataset.objects.create(
            filename=csv_file.name,
            analysis_result=summary
        )

        # Keep only the 5 most recent records
        excess = Dataset.objects.all()[5:]
        if excess.exists():
            excess.delete()

        return Response(summary, status=status.HTTP_200_OK)


class AnalysisHistoryView(APIView):
    permission_classes = [IsAuthenticated]

    def get(self, request):
        datasets = Dataset.objects.all()[:5]
        return Response([
            {
                "filename": d.filename,
                "created_at": d.created_at,
                "summary": d.analysis_result
            }
            for d in datasets
        ])

```

### `analytics/urls.py`

```python
from django.urls import path
from .views import UploadCSVView, AnalysisHistoryView

urlpatterns = [
    path("upload/", UploadCSVView.as_view(), name="upload-csv"),
    path("history/", AnalysisHistoryView.as_view(), name="analysis-history"),
]

```

### `backend/urls.py`

```python
from django.urls import path, include

urlpatterns = [
    path("api/", include("analytics.urls")),
]

```

---

## Web Frontend (React)

Installs:  
```
cd web-frontend
npm install
```

**Key dependencies:**  
- react
- chart.js

### `api.js`

```js
const API_URL = "http://localhost:8000/api/upload/";

export async function uploadCSV(file, token) {
  const formData = new FormData();
  formData.append("file", file);

  const response = await fetch(API_URL, {
    method: "POST",
    headers: {
      Authorization: `Token ${token}`,
    },
    body: formData,
  });

  if (!response.ok) {
    throw new Error("Upload failed");
  }

  return response.json();
}

```

### `App.js`

```js
import Upload from "./components/Upload";

function App() {
  return (
    <div>
      <h2>Chemical Equipment Visualizer</h2>
      <Upload />
    </div>
  );
}

export default App;
```

### `components/Upload.js`

```js
import { useState } from "react";
import { uploadCSV } from "../api";

function Upload() {
  const [result, setResult] = useState(null);

  const handleFileChange = async (event) => {
    const file = event.target.files[0];
    if (!file) return;

    try {
      const data = await uploadCSV(file, "YOUR_TOKEN");
      setResult(data);
    } catch (err) {
      alert(err.message);
    }
  };

  return (
    <div>
      <input type="file" accept=".csv" onChange={handleFileChange} />

      {result && (
        <div>
          <p>Total Equipment: {result.equipment_count}</p>
          <p>Avg Flowrate: {result.average_flowrate.toFixed(2)}</p>
          <p>Avg Pressure: {result.average_pressure.toFixed(2)}</p>
          <p>Avg Temperature: {result.average_temperature.toFixed(2)}</p>
        </div>
      )}
    </div>
  );
}

export default Upload;

```

---

## Desktop Frontend (PyQt5)

Install requirements:
```
cd desktop-frontend
pip install -r requirements.txt
```

**Key dependencies:**  
- pyqt5
- requests
- matplotlib

### `main.py`

```python
import sys
import requests
import matplotlib.pyplot as plt

from PyQt5.QtWidgets import (
    QApplication,
    QWidget,
    QPushButton,
    QFileDialog,
    QMessageBox,
)

API_URL = "http://localhost:8000/api/upload/"
AUTH_HEADER = {"Authorization": "Token YOUR_TOKEN"}


class EquipmentApp(QWidget):
    def __init__(self):
        super().__init__()
        self.setWindowTitle("Chemical Equipment Visualizer")

        self.upload_button = QPushButton("Upload CSV", self)
        self.upload_button.clicked.connect(self.upload_csv)

        self.resize(300, 100)
        self.show()

    def upload_csv(self):
        file_path, _ = QFileDialog.getOpenFileName(
            self, "Select CSV File", "", "CSV Files (*.csv)"
        )

        if not file_path:
            return

        try:
            with open(file_path, "rb") as file:
                response = requests.post(
                    API_URL,
                    files={"file": file},
                    headers=AUTH_HEADER,
                )

            if response.status_code != 200:
                raise RuntimeError(response.text)

            data = response.json()
            self.show_chart(data["equipment_types"])

        except Exception as error:
            QMessageBox.critical(self, "Error", str(error))

    def show_chart(self, distribution):
        labels = list(distribution.keys())
        values = list(distribution.values())

        plt.bar(labels, values)
        plt.xlabel("Equipment Type")
        plt.ylabel("Count")
        plt.title("Equipment Distribution")
        plt.show()


if __name__ == "__main__":
    app = QApplication(sys.argv)
    window = EquipmentApp()
    sys.exit(app.exec_())

```

---

## Sample Data

A sample CSV (`sample_equipment_data.csv`) should be available in the root for testing/demo.

## Usage

1. **Start backend:**  
   ```
  python manage.py runserver
   ```
2. **Run web frontend:**  
   ```
 npm start
   ```
3. **Run desktop frontend:**  
   ```
   python main.py
   ```
4. **Upload CSV files:**  
   Use the web or desktop UI, authenticate via Token, view summary and charts.

## Notes

- Default API assumes API token auth (update `"YOUR_TOKEN"` in examples).
- Only the 5 most recent dataset summaries are stored.
- Update `CORS`/auth settings for production usage as needed.

---

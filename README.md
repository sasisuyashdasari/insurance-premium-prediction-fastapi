# Insurance Premium Prediction API

A FastAPI service that predicts an insurance premium category (for example `Low`, `Medium`, `High`) from a user's basic health, income, occupation and location details. The model is a scikit-learn classifier stored in `model/model.pkl`.

## Project structure

```
.
├── app.py                        # FastAPI app and routes
├── config/
│   └── city_tier.py              # City lists used to assign a city tier (1, 2 or 3)
├── model/
│   ├── model.pkl                 # Trained classifier
│   └── predict.py                # Loads the model and runs predictions
├── schema/
│   ├── user_input.py             # Request validation and derived features (BMI, age group, etc.)
│   └── prediction_response.py    # Response model
├── requirements.txt              # Python dependencies (pinned)
├── Dockerfile                    # Container build
└── .dockerignore                 # Files excluded from the Docker image
```

## Requirements

- Python 3.14 (the pinned packages are built for 3.14)
- Or Docker, if you prefer to run the app in a container

## Run locally

```bash
python -m venv myenv
myenv\Scripts\activate          # Windows
# source myenv/bin/activate     # macOS / Linux

pip install -r requirements.txt
uvicorn app:app --reload
```

Run the command from the project root, because the model is loaded with the relative path `model/model.pkl`.

The API is then available at:

- http://127.0.0.1:8000 (welcome message)
- http://127.0.0.1:8000/docs (interactive API docs)

## Run with Docker

Build and run the image:

```bash
docker build -t insurance-premium-prediction .
docker run -p 8000:8000 insurance-premium-prediction
```

Or pull the published image from Docker Hub:

```bash
docker run -p 8000:8000 axtestingdocket/insurance-premium-prediction:latest
```

## API endpoints

| Method | Path       | Description                                  |
|--------|------------|----------------------------------------------|
| GET    | `/`        | Welcome message                              |
| GET    | `/health`  | Service status, model version, model loaded  |
| POST   | `/predict` | Predict the premium category for one person  |

### `POST /predict`

Request body:

| Field         | Type    | Rules                                                                                          |
|---------------|---------|------------------------------------------------------------------------------------------------|
| `age`         | integer | 1 to 119                                                                                       |
| `weight`      | number  | Greater than 0 (kg)                                                                            |
| `height`      | number  | Greater than 0 and less than 2.5 (m)                                                           |
| `income_lpa`  | number  | Greater than 0 (annual income in lakhs per annum)                                              |
| `smoker`      | boolean | `true` or `false`                                                                              |
| `city`        | string  | Any city name. Leading/trailing spaces are removed and the name is title-cased. Cities not in the lists are treated as tier 3. |
| `occupation`  | string  | One of `retired`, `freelancer`, `student`, `government_job`, `business_owner`, `unemployed`, `private_job` |

The API derives `bmi`, `lifestyle_risk`, `age_group` and `city_tier` from these fields before prediction.

Example request:

```bash
curl -X POST http://127.0.0.1:8000/predict \
  -H "Content-Type: application/json" \
  -d '{"age": 30, "weight": 70, "height": 1.7, "income_lpa": 12, "smoker": false, "city": "Mumbai", "occupation": "private_job"}'
```

Example response:

```json
{
  "response": {
    "predicted_category": "Low",
    "confidence": 0.78,
    "class_probabilities": {"High": 0.01, "Low": 0.78, "Medium": 0.21}
  }
}
```

Invalid input returns HTTP `422` with details about which field failed validation.

## Version

Model version is set in `model/predict.py` (`MODEL_VERSION`) and reported by `/health`.

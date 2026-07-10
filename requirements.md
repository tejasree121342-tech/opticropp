# Requirements Document

## Introduction

OptiCrop is a production-grade agricultural AI platform that predicts the optimal crop to cultivate based on soil chemistry and environmental inputs. The system executes a structured machine learning pipeline — from raw CSV ingestion through model training, evaluation, and best-model selection — and exposes predictions through a Flask REST backend. A professional multi-page frontend presents crop recommendations, suitability analysis, analytics dashboards, and prediction history to end users including farmers, agronomists, and agricultural administrators.

OptiCrop is designed for deployment via Docker and Gunicorn, enforces clean architecture with strict separation of concerns, and meets production-grade security, privacy, and quality standards.

---

## Glossary

- **System**: The OptiCrop application as a whole.
- **ML_Pipeline**: The ordered sequence of data ingestion, preprocessing, training, evaluation, best-model selection, and serialization steps.
- **Preprocessor**: The module responsible for EDA, missing value handling, duplicate removal, outlier treatment, feature scaling, feature engineering, correlation analysis, and class distribution analysis.
- **Trainer**: The module responsible for training each candidate ML algorithm and invoking the Evaluator.
- **Evaluator**: The module responsible for cross-validation, hyperparameter tuning, and metric computation.
- **Model_Selector**: The module responsible for comparing evaluated models and selecting the best one based on F1-weighted score.
- **Model_Registry**: The file-system store (`saved_models/`) where serialized `.pkl` model artifacts are persisted and retrieved.
- **Prediction_Engine**: The module that loads a serialized model from the Model_Registry and produces crop predictions with confidence scores; architecturally separate from Flask routes.
- **Recommendation_Service**: The service layer that invokes the Prediction_Engine for ML-based crop recommendations.
- **Suitability_Service**: The service layer that applies threshold-based environmental rules to determine crop suitability; architecturally separate from the Recommendation_Service.
- **Prediction_Repository**: The data-access layer that persists and retrieves prediction history records in SQLite.
- **Prediction_Record**: A row in the prediction history table containing timestamp, input values, predicted crop, confidence score, model used, and hashed user IP.
- **Input_Vector**: The set of seven numerical features provided by the user: Nitrogen (N), Phosphorous (P), Potassium (K), Temperature (°C), Humidity (%), Rainfall (mm), and pH.
- **F1_Weighted**: The weighted-average F1 score across all crop classes, used as the primary model selection metric.
- **Flask_App**: The Flask web application that exposes HTTP routes, serves templates, and delegates business logic to service layers.
- **Frontend**: The HTML/CSS/JavaScript/Bootstrap 5 user interface served by the Flask_App.
- **Admin_Dashboard**: The restricted administrative interface for model management, prediction log review, and system health monitoring.
- **CSRF_Token**: A cryptographic token used to protect state-changing HTTP requests against cross-site request forgery.
- **Rate_Limiter**: The component that enforces per-IP request rate limits on prediction endpoints.
- **Logger**: The component responsible for structured application logging to file and/or stdout.
- **Config**: The configuration module that loads all settings from environment variables; no hardcoded values are permitted.
- **Dataset**: The CSV file(s) stored in `datasets/` containing labeled agricultural training data.
- **Hashed_IP**: A SHA-256 digest of the raw client IP address stored in place of the raw IP.
- **Docker_Compose**: The container orchestration configuration used to build and run the System in isolated containers.
- **Gunicorn**: The WSGI HTTP server used to serve the Flask_App in production.

---

## Requirements

---

### Requirement 1: CSV Dataset Ingestion

**User Story:** As a data engineer, I want the system to ingest a CSV dataset from a configurable path, so that the ML pipeline always operates on the correct, version-controlled training data.

#### Acceptance Criteria

1. THE ML_Pipeline SHALL read the training Dataset from the file path specified in Config, not from a hardcoded path.
2. IF the Config dataset path is absent or empty, THEN THE ML_Pipeline SHALL raise a descriptive error identifying the missing configuration key and halt execution before attempting to read any file.
3. WHEN the Dataset file is not found at the configured path, THE ML_Pipeline SHALL raise an error whose message includes the configured path and halt execution.
4. WHEN the Dataset file is found, THE ML_Pipeline SHALL load it into a Pandas DataFrame and pass it to the Preprocessor.
5. IF the Dataset contains zero rows or zero columns after loading, THEN THE ML_Pipeline SHALL raise a descriptive error stating the actual row and column counts and halt execution.
6. IF the Dataset file cannot be parsed as valid CSV, THEN THE ML_Pipeline SHALL raise a descriptive parse error distinct from the file-not-found error and halt execution.
7. THE ML_Pipeline SHALL log the number of rows and columns loaded from the Dataset.

---

### Requirement 2: Data Preprocessing

**User Story:** As a data scientist, I want all preprocessing steps executed in a defined order by a dedicated module, so that data quality is guaranteed before model training begins.

#### Acceptance Criteria

1. THE Preprocessor SHALL execute preprocessing steps in this order: missing value handling, duplicate removal, outlier treatment, feature scaling, feature engineering, correlation analysis, and class distribution analysis.
2. WHEN missing values are detected in the Dataset, THE Preprocessor SHALL impute or drop them according to the strategy defined in Config, and SHALL log the count of affected rows.
3. WHEN duplicate rows are detected — where a duplicate row is defined as a row with identical values across all columns — THE Preprocessor SHALL remove them and SHALL log the count removed.
4. WHEN numerical feature values fall outside the bounds defined in Config, THE Preprocessor SHALL treat them as outliers using the configured strategy and SHALL log the count treated.
5. THE Preprocessor SHALL apply feature scaling to all numerical Input_Vector features using the scaler type defined in Config.
6. THE Preprocessor SHALL compute and log the Pearson correlation matrix for all numerical features.
7. THE Preprocessor SHALL compute and log the class distribution for the target label column, recording both the count and relative frequency for each class.
8. THE Preprocessor SHALL return a fully preprocessed DataFrame and the fitted scaler artifact to the ML_Pipeline.
9. THE Preprocessor SHALL never import or reference Flask routes or prediction logic.
10. IF Config does not define a valid strategy for any required preprocessing step (missing value handling, outlier treatment, or feature scaling), THEN THE Preprocessor SHALL raise a descriptive error identifying the missing strategy key and halt execution.

---

### Requirement 3: Train/Test Split

**User Story:** As a data scientist, I want the preprocessed dataset split into training and test sets with a configurable ratio, so that model evaluation reflects performance on unseen data.

#### Acceptance Criteria

1. THE ML_Pipeline SHALL split the preprocessed Dataset into training and test sets using the test ratio defined in Config, where the ratio must be a float in the exclusive range (0.0, 1.0).
2. IF the configured test ratio is outside the exclusive range (0.0, 1.0), THEN THE ML_Pipeline SHALL raise a descriptive error stating the invalid value and halt execution before splitting.
3. THE ML_Pipeline SHALL apply stratified splitting to preserve class distribution across both sets; IF any class has fewer than 2 samples and stratified splitting cannot be applied, THEN THE ML_Pipeline SHALL log a warning and fall back to non-stratified splitting.
4. THE ML_Pipeline SHALL set a random seed from Config, where the seed must be an integer in the range [0, 2^32 − 1], to ensure reproducibility.
5. THE ML_Pipeline SHALL log the row count for the training set and the row count for the test set after splitting.

---

### Requirement 4: Multi-Algorithm Model Training

**User Story:** As a data scientist, I want all nine candidate algorithms trained on the same training set, so that they can be objectively compared.

#### Acceptance Criteria

1. THE Trainer SHALL train the following nine algorithms: Logistic Regression, Decision Tree, Random Forest, K-Nearest Neighbors, Naive Bayes, Support Vector Machine, Gradient Boosting, Extra Trees, and XGBoost.
2. THE Trainer SHALL train each algorithm independently on the same training set produced by the ML_Pipeline, ensuring no fitted state is shared between algorithms.
3. WHEN training of an algorithm completes successfully, THE Trainer SHALL pass that trained model to the Evaluator for metric computation.
4. THE Trainer SHALL never mix preprocessing logic or Flask route logic into training code.
5. WHEN the Trainer begins training an algorithm, THE Trainer SHALL log the algorithm name and a start timestamp.
6. WHEN the Trainer finishes training an algorithm, THE Trainer SHALL log the algorithm name and a completion timestamp.
7. IF the training set produced by the ML_Pipeline is empty or unavailable, THEN THE Trainer SHALL halt all training and log an error indicating the training set could not be loaded.
8. IF an individual algorithm fails to train, THEN THE Trainer SHALL log an error identifying the algorithm name and the reason for failure, skip that algorithm, and continue training the remaining algorithms.

---

### Requirement 5: Model Evaluation with Cross-Validation and Hyperparameter Tuning

**User Story:** As a data scientist, I want each trained model evaluated with cross-validation and tuned with hyperparameter search, so that reported metrics reflect the best attainable performance per algorithm.

#### Acceptance Criteria

1. THE Evaluator SHALL perform k-fold cross-validation on each model using the fold count defined in Config; IF Config omits the fold count, THE Evaluator SHALL default to k=5.
2. THE Evaluator SHALL execute hyperparameter tuning for each model using the search strategy defined in Config; IF Config omits the search strategy, THE Evaluator SHALL default to grid search with no more than 50 candidate configurations.
3. THE Evaluator SHALL compute the following metrics for each model on the test set using the best hyperparameter configuration found: accuracy, F1-weighted, precision-weighted, recall-weighted, and ROC-AUC (using one-vs-rest averaging for datasets with 10 or fewer distinct class labels).
4. THE Evaluator SHALL record both the mean and standard deviation of F1-weighted scores across CV folds for each model within the evaluation report.
5. THE Evaluator SHALL return an evaluation report containing all metrics for all models to the Model_Selector.
6. IF a model fails cross-validation or hyperparameter tuning, THE Evaluator SHALL record an error entry for that model in the evaluation report and continue evaluating the remaining models.
7. THE Evaluator SHALL never import or reference Flask routes or preprocessing logic.

---

### Requirement 6: Best Model Selection

**User Story:** As a data scientist, I want the model with the highest F1-weighted score selected automatically, so that the deployed model is optimized for multi-class crop prediction rather than raw accuracy.

#### Acceptance Criteria

1. THE Model_Selector SHALL select the model with the highest F1_Weighted score (a decimal value in [0.0, 1.0]) from the evaluation report as the best model.
2. WHEN two or more models share the highest F1_Weighted score, THE Model_Selector SHALL break the tie using precision-weighted score; WHEN precision-weighted scores also tie, THE Model_Selector SHALL select the model whose name is alphabetically first.
3. THE Model_Selector SHALL log the name and F1_Weighted score of the selected best model.
4. THE Model_Selector SHALL log a summary table of all model names and their F1_Weighted scores in descending order.
5. IF the evaluation report contains no models with a valid F1_Weighted score, THEN THE Model_Selector SHALL raise a descriptive error and halt execution.
6. IF one or more models in the evaluation report have missing F1_Weighted scores due to evaluation failure, THE Model_Selector SHALL exclude those models from selection and log their names.
7. THE Model_Selector SHALL confirm in its log output that F1_Weighted was used as the primary selection criterion.

---

### Requirement 7: Model Serialization

**User Story:** As a deployment engineer, I want the best model and its scaler saved as versioned `.pkl` artifacts, so that the Prediction_Engine can load them without retraining.

#### Acceptance Criteria

1. WHEN the best model is selected, THE Model_Selector SHALL serialize it to the Model_Registry using a filename that includes the model name and a UTC ISO 8601 timestamp, at the path defined in Config, using Python's `pickle` module.
2. WHEN the best model is serialized, THE Model_Selector SHALL serialize the fitted scaler artifact to the same Model_Registry directory using a matching versioned filename.
3. THE Model_Selector SHALL record the model name, F1_Weighted score, and serialization timestamp in UTC ISO 8601 format in a JSON metadata file within the Model_Registry.
4. IF the Model_Registry directory does not exist, THEN THE Model_Selector SHALL create it before writing artifacts.
5. IF serialization fails due to an I/O error or permission error, THEN THE Model_Selector SHALL log the error including the target path and re-raise the exception to halt pipeline execution.
6. THE Model_Selector SHALL log the file paths of all serialized artifacts upon successful completion.

---

### Requirement 8: Prediction Engine

**User Story:** As a backend engineer, I want a dedicated Prediction_Engine that loads the serialized model and produces crop predictions, so that prediction logic is never mixed with HTTP routing code.

#### Acceptance Criteria

1. WHEN the Prediction_Engine is initialized, THE Prediction_Engine SHALL load the model artifact designated as the active model in the Config from the Model_Registry.
2. WHEN the Prediction_Engine is initialized, THE Prediction_Engine SHALL load the fitted scaler artifact from the Model_Registry path defined in Config.
3. WHEN an Input_Vector is received, THE Prediction_Engine SHALL apply the loaded scaler to the Input_Vector before inference.
4. WHEN inference completes, THE Prediction_Engine SHALL return a structured result object containing exactly two fields: `predicted_label` (a string crop name) and `confidence_score` (a float in [0.0, 1.0]).
5. IF the model artifact is not found in the Model_Registry at the configured path, THEN THE Prediction_Engine SHALL raise an error on initialization whose message includes the missing file path.
6. IF the scaler artifact is not found in the Model_Registry at the configured path, THEN THE Prediction_Engine SHALL raise an error on initialization whose message includes the missing scaler path.
7. IF an Input_Vector with invalid shape or non-numeric values is passed to inference, THEN THE Prediction_Engine SHALL raise a descriptive error and not return a prediction result.
8. THE Prediction_Engine SHALL never import or reference Flask routes, database access, or preprocessing logic.
9. THE Prediction_Engine SHALL be importable independently of the Flask_App for use in scripts and tests.

---

### Requirement 9: Flask Application Structure

**User Story:** As a backend engineer, I want the Flask application organized into controllers, services, repositories, routes, and models following clean architecture, so that each layer has a single responsibility and can be tested in isolation.

#### Acceptance Criteria

1. THE Flask_App SHALL register all routes via Blueprint objects defined in the `app/routes/` package.
2. THE Flask_App SHALL delegate all business logic to service layer classes in `app/services/`; route handlers SHALL NOT directly execute business logic.
3. THE Flask_App SHALL delegate all database access to repository classes in `app/repositories/`; route handlers and service classes SHALL NOT directly execute SQL or ORM queries.
4. THE Flask_App SHALL load all configuration from Config, which reads from environment variables.
5. THE Flask_App SHALL never contain ML training, preprocessing, or model selection logic in route handlers.
6. WHEN the Flask_App starts, THE Flask_App SHALL initialize the Logger.
7. WHEN the Flask_App receives an HTTP request, THE Flask_App SHALL log the request method and path at INFO level.
8. IF the model artifact does not exist in the Model_Registry at startup, THEN THE Flask_App SHALL log a WARNING identifying the missing artifact path; the application SHALL still start and serve non-prediction routes.

---

### Requirement 10: Crop Recommendation Endpoint

**User Story:** As a farmer, I want to submit soil and environmental readings and receive an ML-based crop recommendation, so that I can make informed planting decisions.

#### Acceptance Criteria

1. WHEN a POST request containing a valid Input_Vector is received at the recommendation endpoint, THE Recommendation_Service SHALL invoke the Prediction_Engine and return the predicted crop label and a confidence score as a decimal in [0.0, 1.0].
2. THE Recommendation_Service SHALL validate that all seven Input_Vector fields are present and within their defined acceptable ranges before invoking the Prediction_Engine.
3. IF any Input_Vector field is missing, THEN THE Flask_App SHALL return a 400 response with a field-level error message identifying the missing field by name.
4. IF any Input_Vector field value is outside its defined acceptable range, THEN THE Flask_App SHALL return a 400 response with a field-level error message identifying the offending field and its valid range.
5. WHEN a valid prediction is produced, THE Recommendation_Service SHALL instruct the Prediction_Repository to persist a Prediction_Record containing the Input_Vector values, predicted crop, confidence score, model name, UTC ISO 8601 timestamp, and Hashed_IP.
6. THE Recommendation_Service SHALL never invoke the Suitability_Service or share its service layer code with the Suitability_Service.
7. IF the Prediction_Engine raises an error during inference after validation passes, THEN THE Flask_App SHALL return a 500 response with a sanitized error message that does not expose internal stack traces or file paths.
8. IF the Prediction_Repository fails to persist the Prediction_Record, THEN THE Flask_App SHALL log the failure at ERROR level and return a 500 response with a sanitized error message.

---

### Requirement 11: Crop Suitability Endpoint

**User Story:** As an agronomist, I want to submit environmental readings and receive a threshold-based suitability analysis for each crop, so that I can evaluate whether conditions are suitable independent of ML predictions.

#### Acceptance Criteria

1. WHEN a POST request containing a valid Input_Vector is received at the suitability endpoint, THE Suitability_Service SHALL evaluate each crop's threshold conditions and return a suitability indicator of "suitable", "marginal", or "unsuitable" for each crop defined in Config.
2. THE Suitability_Service SHALL determine suitability using agronomic threshold rules defined in Config, not using any ML model.
3. THE Suitability_Service SHALL validate that all seven Input_Vector fields are present and within the acceptable ranges defined in Requirement 13 before evaluation.
4. IF any Input_Vector field is missing, THEN THE Flask_App SHALL return a 400 response with a field-level error message identifying the missing field by name.
5. IF any Input_Vector field value is invalid, THEN THE Flask_App SHALL return a 400 response with a field-level error message identifying the offending field and its valid range.
6. THE Suitability_Service SHALL never invoke the Recommendation_Service or the Prediction_Engine.
7. THE Suitability_Service SHALL be implemented in a separate service module from the Recommendation_Service.

---

### Requirement 12: Prediction History Persistence

**User Story:** As a system administrator, I want all predictions stored in SQLite with full input traceability, so that predictions can be audited and reviewed.

#### Acceptance Criteria

1. THE Prediction_Repository SHALL persist each Prediction_Record to a SQLite database at the path defined in Config.
2. THE Prediction_Repository SHALL store the following fields per record: timestamp (UTC ISO 8601), N, P, K, temperature, humidity, rainfall, pH, predicted crop, confidence score (float), model name, and Hashed_IP.
3. THE Prediction_Repository SHALL store the Hashed_IP as the SHA-256 hex digest of the raw client IP address; the raw IP SHALL never be persisted in any column.
4. THE Prediction_Repository SHALL expose a method accepting `page` and `page_size` parameters to retrieve paginated Prediction_Records ordered by timestamp descending.
5. THE Prediction_Repository SHALL expose a method that returns the total count of Prediction_Records as a non-negative integer.
6. IF a database write fails, THEN THE Prediction_Repository SHALL log the error at ERROR level including the exception message and raise a descriptive exception to the caller.

---

### Requirement 13: Input Validation

**User Story:** As a security engineer, I want all user inputs sanitized and validated server-side before use, so that malformed or malicious data cannot reach ML inference or the database.

#### Acceptance Criteria

1. THE Flask_App SHALL validate all form and JSON inputs server-side before passing them to any service layer.
2. IF any Input_Vector field contains a non-numeric value, THEN THE Flask_App SHALL return a 400 response with an error message identifying the field and stating that a numeric value is required.
3. IF any Input_Vector field value falls outside the following ranges, THEN THE Flask_App SHALL return a 400 response identifying the field and its valid range: N ∈ [0, 140] kg/ha, P ∈ [5, 145] kg/ha, K ∈ [5, 205] kg/ha, Temperature ∈ [0, 50] °C, Humidity ∈ [0, 100] %, Rainfall ∈ [0, 300] mm, pH ∈ [0, 14].
4. THE Flask_App SHALL sanitize all string inputs to remove HTML special characters before logging or storing.
5. THE Frontend SHALL apply client-side JavaScript validation that mirrors the same numeric-type and range rules as the server-side validation, blocking form submission and displaying inline field-level error messages before any HTTP request is sent.

---

### Requirement 14: CSRF Protection

**User Story:** As a security engineer, I want all state-changing form submissions protected with CSRF tokens, so that cross-site request forgery attacks are mitigated.

#### Acceptance Criteria

1. THE Flask_App SHALL enable CSRF protection for all POST, PUT, PATCH, and DELETE routes.
2. THE Flask_App SHALL embed a CSRF_Token in every HTML form rendered by the Frontend as a hidden input field.
3. IF a state-changing request arrives without a valid CSRF_Token, THEN THE Flask_App SHALL reject it with a 403 response and not process the request body.
4. THE Flask_App SHALL source the CSRF secret from an environment variable defined in Config; if the variable is absent at startup, the application SHALL halt per Requirement 16.

---

### Requirement 15: Rate Limiting

**User Story:** As a security engineer, I want prediction endpoints rate-limited per IP, so that automated abuse and denial-of-service attempts are mitigated.

#### Acceptance Criteria

1. THE Rate_Limiter SHALL enforce an independent per-IP request limit on the crop recommendation endpoint using the threshold and window duration defined in Config.
2. THE Rate_Limiter SHALL enforce an independent per-IP request limit on the crop suitability endpoint using the threshold and window duration defined in Config.
3. WHEN a client's request count is within the configured limit for the current window, THE Flask_App SHALL process the request normally.
4. WHEN a client exceeds the configured rate limit, THE Flask_App SHALL return a 429 response with a message indicating the rate limit has been exceeded and a `Retry-After` header specifying the remaining window duration in seconds.
5. WHEN the rate-limit window expires, THE Rate_Limiter SHALL reset the request count for that IP to zero.
6. THE Rate_Limiter SHALL source all limit thresholds and window durations from Config.

---

### Requirement 16: Security Configuration

**User Story:** As a security engineer, I want all secrets and configuration values sourced from environment variables, so that no sensitive values are hardcoded in source code.

#### Acceptance Criteria

1. THE Config SHALL load the Flask secret key, database path, model paths, CSRF secret, rate limit thresholds, and all other configurable values exclusively from environment variables; the canonical list of required variables SHALL be defined in the `.env.example` file.
2. THE Flask_App SHALL provide a `DevelopmentConfig` class with debug mode enabled and a `ProductionConfig` class with debug mode disabled, selected by the `FLASK_ENV` environment variable.
3. IF a required environment variable listed in `.env.example` is absent at startup, THEN THE Flask_App SHALL log an error message identifying the missing variable name, halt without serving any requests, and exit with a non-zero status code.
4. THE System SHALL provide a `.env.example` file that lists all required environment variable names without values, and marks optional variables as optional with their default values.
5. THE System SHALL include a `.gitignore` entry that excludes any `.env` file from version control, ensuring no file containing real secrets is committed.

---

### Requirement 17: Logging and Exception Handling

**User Story:** As a DevOps engineer, I want structured logging and centralized exception handling, so that errors can be diagnosed in production without exposing sensitive information to end users.

#### Acceptance Criteria

1. THE Logger SHALL write structured log entries including timestamp, severity level, module name, and message to the log file path defined in Config; IF the configured log path is inaccessible at startup, THE Logger SHALL fall back to stdout and log a warning about the fallback.
2. THE Flask_App SHALL register a global exception handler that catches unhandled exceptions, logs the full stack trace at ERROR level with a unique correlation ID, and returns a sanitized error response that excludes stack traces, internal file paths, environment variable values, and database connection details.
3. WHEN an unhandled exception occurs, THE Flask_App SHALL return the 500 error page to the client; the response body SHALL contain the correlation ID to aid diagnosis without exposing internals.
4. WHEN a prediction request is processed, THE Flask_App SHALL log at INFO level the Input_Vector field values and the predicted crop result, and SHALL NOT include the raw client IP address in the log entry.
5. THE Logger SHALL never write raw client IP addresses to log files or stdout.

---

### Requirement 18: Frontend — Home Page

**User Story:** As a visitor, I want a modern landing page that introduces OptiCrop, so that I can understand the platform's value and navigate to its features.

#### Acceptance Criteria

1. THE Frontend SHALL render a Home page at the root URL (`/`) that includes a hero section, a feature highlights section, and a call-to-action button that navigates to the Crop Recommendation page at `/recommend`.
2. THE Frontend SHALL apply Bootstrap 5 responsive layout classes so that the Home page renders without horizontal overflow or hidden content on viewports from 320px to 1920px wide.
3. WHEN the Home page DOM content is loaded, THE Frontend SHALL apply CSS animation classes to hero and feature elements so they transition from hidden to visible within 800ms.
4. THE Frontend SHALL support both dark mode and light mode; the active mode SHALL be determined by the value stored in `localStorage` under the key `opticrop-theme`, defaulting to light mode if no value is stored; a toggle control SHALL update both the active mode and the stored preference.

---

### Requirement 19: Frontend — Crop Recommendation Page

**User Story:** As a farmer, I want an input form where I can enter my soil and environmental readings and receive a crop recommendation with a confidence score, so that I can act on the ML prediction immediately.

#### Acceptance Criteria

1. THE Frontend SHALL render a Crop Recommendation page that presents a form with labeled inputs for all seven Input_Vector fields, each displaying its unit and valid range as hint text.
2. THE Frontend SHALL perform client-side validation on all Input_Vector fields before submission, blocking the request and displaying inline field-level error messages for non-numeric or out-of-range values.
3. WHEN the recommendation form is submitted, THE Frontend SHALL display a loading indicator and disable the submit button until the response is received.
4. WHEN a successful recommendation response is received, THE Frontend SHALL display the predicted crop name, the confidence score formatted as a percentage rounded to one decimal place, and agronomic notes for the recommended crop.
5. WHEN a 400 validation error response is received, THE Frontend SHALL display the field-level error messages returned by the Flask_App inline next to the corresponding fields.
6. WHEN a 500 server error response is received, THE Frontend SHALL display a user-friendly error alert containing the correlation ID from the response, without exposing stack traces or internal details.

---

### Requirement 20: Frontend — Crop Suitability Page

**User Story:** As an agronomist, I want a dedicated page for threshold-based suitability analysis that is visually and architecturally distinct from the recommendation page, so that the two analysis modes are never conflated.

#### Acceptance Criteria

1. THE Frontend SHALL render a Crop Suitability page that presents a form with labeled inputs for all seven Input_Vector fields, each displaying its unit and valid range as hint text.
2. WHEN suitability results are returned, THE Frontend SHALL display the crops grouped by suitability tier — "suitable", "marginal", "unsuitable" — with all evaluated crops appearing in exactly one tier.
3. THE Frontend SHALL use a distinct color scheme and page heading for the Crop Suitability page that differs from the Crop Recommendation page, such that no element color or title is shared between the two pages.
4. THE Frontend SHALL submit suitability requests to the suitability endpoint, not to the recommendation endpoint.
5. WHEN the suitability form is submitted, THE Frontend SHALL display a loading indicator and disable the submit button until the response is received.
6. WHEN a validation or server error response is received, THE Frontend SHALL display the error message returned by the Flask_App without exposing technical details.

---

### Requirement 21: Frontend — Dashboard Page

**User Story:** As an administrator, I want a professional analytics dashboard that shows prediction activity, model performance, and crop distribution at a glance, so that I can monitor system usage.

#### Acceptance Criteria

1. THE Frontend SHALL render a Dashboard page that displays: total prediction count, most recommended crop, average confidence score as a percentage rounded to one decimal place, and a prediction activity chart covering the last 30 days.
2. THE Frontend SHALL render all Dashboard charts using a JavaScript charting library whose source URL is defined in Config.
3. THE Frontend SHALL apply Bootstrap 5 responsive grid layout to all Dashboard widgets so that the layout renders without overflow or clipping at the xs, sm, md, lg, and xl breakpoints.
4. WHEN Dashboard data is loading, THE Frontend SHALL display skeleton loading states for each widget; WHEN data has loaded or failed, THE Frontend SHALL replace the skeleton with the data or an error message.
5. IF a Dashboard data fetch fails, THE Frontend SHALL display an error message for the affected widget(s) while preserving the display of any widget data that loaded successfully.
6. IF the charting library fails to load, THE Frontend SHALL display an unavailability message in place of each affected chart widget.

---

### Requirement 22: Frontend — Analytics Page

**User Story:** As a data analyst, I want an analytics page with model comparison charts and prediction trend visualizations, so that I can evaluate model performance and usage patterns.

#### Acceptance Criteria

1. THE Frontend SHALL render an Analytics page displaying a bar chart comparing F1_Weighted scores across all trained models for which metrics are available.
2. THE Frontend SHALL render a line or area chart showing daily prediction volume over time using all available historical prediction records.
3. THE Frontend SHALL render a pie or donut chart showing the distribution of predicted crops from all available historical prediction records.
4. WHEN chart data is loading, THE Frontend SHALL display a loading indicator for each chart.
5. WHEN chart data is unavailable or the fetch fails, THE Frontend SHALL display a descriptive placeholder message stating "Data unavailable" in place of each affected chart rather than rendering an empty or broken chart.

---

### Requirement 23: Frontend — Research Page

**User Story:** As a researcher, I want a Research page that presents EDA visualizations and dataset insights, so that I can understand the underlying data distribution and feature relationships.

#### Acceptance Criteria

1. THE Frontend SHALL render a Research page that displays pre-generated EDA visualizations: a correlation heatmap, a feature distribution histogram for each dataset feature, and a class distribution chart.
2. WHEN ML_Pipeline training completes, THE ML_Pipeline SHALL generate and save all EDA visualization images to the configured static assets directory so that the Frontend can serve them without runtime chart computation.
3. IF any EDA visualization image is missing or fails to load, THE Frontend SHALL render a descriptive placeholder in its place rather than a broken image element.
4. THE Frontend SHALL display dataset summary statistics including total row count, total column count, and per-class sample count for each target class label.
5. IF dataset summary statistics are unavailable, THE Frontend SHALL display an error message stating that statistics could not be loaded rather than rendering empty cells.

---

### Requirement 24: Frontend — Prediction History Page

**User Story:** As a user, I want to view a paginated history of predictions with input values and confidence scores, so that I can review past recommendations.

#### Acceptance Criteria

1. THE Frontend SHALL render a Prediction History page that displays Prediction_Records in a paginated table ordered by timestamp descending.
2. THE Frontend SHALL display the following columns: timestamp, N, P, K, temperature, humidity, rainfall, pH, predicted crop, confidence score, and model name.
3. THE Prediction History page SHALL NOT display raw IP addresses or Hashed_IPs to end users.
4. WHEN no prediction history exists, THE Frontend SHALL display an informative empty-state message.
5. WHEN prediction history data is being fetched, THE Frontend SHALL display a loading indicator and disable pagination controls until the data is available.

---

### Requirement 25: Frontend — Admin Dashboard

**User Story:** As a system administrator, I want an Admin Dashboard restricted to authorized users that provides model management, prediction log review, and system health metrics, so that I can operate and audit the platform.

#### Acceptance Criteria

1. IF a request to the Admin_Dashboard is made by a user who is not authenticated with the administrator role, THEN THE Admin_Dashboard SHALL redirect the request to the login page with a 302 HTTP response.
2. THE Admin_Dashboard SHALL display the current active model name, its F1_Weighted score as a decimal between 0.00 and 1.00, and its serialization timestamp in UTC ISO 8601 format, sourced from the Model_Registry metadata file.
3. WHEN an administrator triggers the retraining control, THE Admin_Dashboard SHALL invoke the ML_Pipeline as a background task and display a status indicator showing one of the following states: pending, in-progress, completed, or failed.
4. IF the ML_Pipeline retraining task fails, THEN THE Admin_Dashboard SHALL display an error message indicating that retraining failed; the active model SHALL remain unchanged and the dashboard SHALL reflect the previously active model.
5. THE Admin_Dashboard SHALL display a paginated log of the 500 most recent Prediction_Records at a maximum of 25 records per page, showing all fields except raw IP addresses; Hashed_IPs SHALL be displayed in place of raw IPs.
6. THE Admin_Dashboard SHALL display system health metrics: application uptime in days, hours, and minutes; total prediction count as a non-negative integer; database file size in megabytes rounded to two decimal places; and last training timestamp in UTC ISO 8601 format.

---

### Requirement 26: Frontend — Additional Pages

**User Story:** As a visitor, I want an About page and a Contact page so that I can learn about the platform and get in touch.

#### Acceptance Criteria

1. THE Frontend SHALL render an About page at `/about` that describes the platform's purpose, ML methodology, and technology stack including all libraries listed in `requirements.txt`.
2. THE Frontend SHALL render a Contact page at `/contact` with a form containing at minimum a name field, an email field, and a message field, all marked as required.
3. IF the Contact form is submitted with any required field empty, THEN THE Frontend SHALL display a client-side validation error next to the empty field and SHALL NOT submit the form or send an HTTP request.

---

### Requirement 27: Error Pages

**User Story:** As a user, I want informative custom error pages for 404 and 500 errors, so that I am never left staring at a raw browser error screen.

#### Acceptance Criteria

1. WHEN a request is made for a route that does not exist, THE Flask_App SHALL return an HTTP 404 response rendering a custom HTML page that includes a message indicating the page was not found and a "Home" link that navigates to the root URL `/`.
2. WHEN an unhandled server error occurs, THE Flask_App SHALL return an HTTP 500 response rendering a custom HTML page that includes a message indicating a server error occurred and a "Home" link that navigates to the root URL `/`.
3. THE 404 and 500 error page templates SHALL extend the shared base layout template used by the rest of the Frontend, ensuring consistent Bootstrap 5 theming.

---

### Requirement 28: Testing Suite

**User Story:** As a quality engineer, I want a comprehensive Pytest suite covering routes, predictions, validation, and database operations, so that regressions are caught before deployment.

#### Acceptance Criteria

1. THE System SHALL include a Pytest test suite located in the `tests/` directory; each test module SHALL use a dedicated in-memory SQLite database and a mock Prediction_Engine to remain isolated from production state.
2. THE System SHALL include route tests that assert correct HTTP status codes and JSON response shapes for the recommendation endpoint, suitability endpoint, and prediction history endpoint.
3. THE System SHALL include prediction tests that verify the Prediction_Engine returns a `predicted_label` string and a `confidence_score` float in [0.0, 1.0] for any valid Input_Vector.
4. THE System SHALL include validation tests that assert a 400 response containing a field-level error message is returned for each invalid Input_Vector case: missing field, non-numeric field, and out-of-range field.
5. THE System SHALL include database tests that verify: a Prediction_Record is persisted with all fields matching the values passed to the repository; records are retrieved in descending timestamp order; and the count method returns the correct total.
6. WHEN tests are run with the `--cov` flag, THE System SHALL produce a coverage report; the branch coverage across all production source modules SHALL be 80% or above.
7. FOR ALL valid Input_Vector inputs sampled across the full valid range of each field, THE Prediction_Engine SHALL return a `predicted_label` that belongs to the set of crop labels present in the training Dataset.
8. FOR ALL valid Input_Vector inputs, applying the fitted scaler followed by the inverse transform SHALL return values within 1e-6 absolute tolerance of the original input values.

---

### Requirement 29: Docker and Deployment

**User Story:** As a DevOps engineer, I want the application containerized with Docker Compose and served by Gunicorn, so that it can be deployed consistently across Linux, Windows, and macOS environments.

#### Acceptance Criteria

1. THE System SHALL include a `Dockerfile` that builds the Flask_App image using the official Python 3.11 base image and installs all dependencies from a pinned `requirements.txt`.
2. THE System SHALL include a `docker-compose.yml` that defines a service for the Flask_App, mounts the `datasets/`, `saved_models/`, `logs/`, and `instance/` directories as named volumes, and exposes the application on port 5000.
3. WHEN the container starts with `FLASK_ENV=production`, THE Flask_App SHALL be served by Gunicorn with a worker count sourced from Config, defaulting to 4 workers if Config omits the value, with a maximum of 8 workers.
4. THE System SHALL select `DevelopmentConfig` when `FLASK_ENV=development` and `ProductionConfig` when `FLASK_ENV=production`; IF `FLASK_ENV` is absent, THE System SHALL default to `ProductionConfig`.
5. THE System SHALL include a `requirements.txt` that pins all dependency versions using exact version specifiers (`==`) including XGBoost.
6. WHEN `docker-compose up` is executed on a machine with Docker and Docker Compose installed, THE System SHALL start and return an HTTP 200 response on the health check route within 30 seconds.
7. IF `FLASK_ENV` is absent from the container environment, THE System SHALL default to `ProductionConfig` and log a warning that the environment was not explicitly set.

---

### Requirement 30: Documentation

**User Story:** As a developer or operator, I want comprehensive documentation covering installation, architecture, API, folder structure, and deployment, so that I can understand, operate, and extend the platform.

#### Acceptance Criteria

1. THE System SHALL include a `README.md` at the project root with an overview, prerequisites, quick-start instructions for local and Docker deployments, and links to all documentation files in `docs/`.
2. THE System SHALL include a `docs/installation.md` file with step-by-step installation instructions for local virtual environment and Docker deployments, including all required environment variable setup steps.
3. THE System SHALL include a `docs/architecture.md` file describing the clean architecture layers, each module's responsibility, and the end-to-end data flow through the ML_Pipeline and Flask_App.
4. THE System SHALL include a `docs/api.md` file documenting all HTTP endpoints with method, URL, accepted request body format, success response schema, and all possible error response codes and messages.
5. THE System SHALL include a `docs/deployment.md` file describing production deployment steps including environment variable configuration, Gunicorn worker tuning, and log rotation.
6. THE System SHALL include a `docs/docker.md` file describing Docker image build, container run, volume configuration, port mapping, and Docker Compose networking.

---

### Requirement 31: Non-Functional Requirements

**User Story:** As a product owner, I want the system to meet production-grade quality standards for performance, maintainability, accessibility, and code hygiene, so that it operates reliably at scale and can be maintained long-term.

#### Acceptance Criteria

1. THE System SHALL enforce clean architecture by ensuring that no ML training or preprocessing module is imported in any Flask route module.
2. THE System SHALL enforce clean architecture by ensuring that no Flask route module or database repository is imported in any ML pipeline module.
3. THE System SHALL contain no hardcoded file paths, secrets, thresholds, or configuration values outside of Config.
4. THE System SHALL contain no TODO comments, placeholder functions, or pseudo-code in any production source file.
5. THE Frontend SHALL achieve a Lighthouse accessibility score of 80 or above on the Home and Crop Recommendation pages.
6. WHEN the prediction endpoint receives a valid Input_Vector with no concurrent requests in flight and the model already loaded into memory, THE Flask_App SHALL return a complete HTTP response within 2000 milliseconds measured from request receipt to response sent.
7. THE Frontend SHALL render without horizontal scroll bars, clipped content, or inoperable interactive controls at any viewport width from 320px to 1920px.
8. THE Frontend SHALL support dark mode and light mode, with the user's preference persisted in `localStorage` under the key `opticrop-theme` across sessions.
9. THE System SHALL use Bootstrap Icons exclusively for all iconography in the Frontend; no other icon library SHALL be loaded.
10. THE System SHALL include no module exceeding 300 lines of production code and no function exceeding 50 lines; each module and each function SHALL have a single entry point and a single clearly defined responsibility.

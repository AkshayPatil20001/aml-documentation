# Comprehensive Interview Preparation Guide: AML Detection System
**100 Deep-Dive Project Questions (Zero Framework/Django, 100% Core Logic & Architecture)**

---

## Part 1: General Project & AML Domain (Q1 - Q15)
**1. What is the fundamental problem your AML Detection System solves?**
**Answer:** Traditional AML systems rely on rigid "boolean rules" (e.g., flagging any transaction over ₹50k). This causes a 95% false positive rate and misses complex laundering. My project replaces rules with an unsupervised Machine Learning model (Isolation Forest) to dynamically detect behavioral anomalies, and uses a Retrieval-Augmented Generation (RAG) AI to auto-draft the corresponding Suspicious Activity Reports (SARs).

**2. What is a False Positive in AML?**
**Answer:** When a legitimate customer is halted or flagged as suspicious merely because they triggered a static threshold (like depositing payroll just over a limit). This wastes compliance analysts' time.

**3. What is "Structuring" (Smurfing)?**
**Answer:** A taxonomy of laundering where a criminal breaks a large sum (e.g., ₹2L) into smaller, sub-threshold deposits (four deposits of ₹49k) to evade automatic regulatory reporting.

**4. How does your system detect Structuring?**
**Answer:** Instead of looking at single transactions, we use Pandas to group transactions within a 7-day rolling window, counting the frequency of deposits that sit immediately below the legal ₹50k threshold (e.g., ₹40k-₹49k bracket).

**5. What is a "Money Mule"?**
**Answer:** An account used explicitly to receive illicit funds and rapidly move them to another central account, masking the origin. 

**6. How is a Money Mule detected computationally?**
**Answer:** By calculating a "Velocity Score"—the mathematical ratio between total incoming funds and outgoing funds within an ultra-short timeframe (e.g., 24 hours), combined with a resting balance that drops back to zero.

**7. What is "Round-Tripping"?**
**Answer:** Creating fake financial volume by sending money out to a counterparty and having the exact amount returned shortly after. The net change is zero, but it creates a trail of "legitimate" looking business volume.

**8. How do you detect Round-Tripping?**
**Answer:** By vectorizing directional flows between counterparties. We calculate the absolute variance between `sum(inflow)` and `sum(outflow)` for a specific account pairing. Near-zero variance with high volume triggers a round-trip flag.

**9. Why do you use both Machine Learning AND Rule-Based tags?**
**Answer:** ML mathematically identifies *that* an account is anomalous, but it can't explain *why* to a judge. "Black box" AI isn't legally permissible alone. We use ML for detection, and rule-based vectors for "Explainability/Tagging" (telling the analyst if the anomaly looks like Structuring or Muling).

**10. Walk me through the data pipeline.**
**Answer:** Raw CSV ingest → Pandas schema normalization → Type casting & cleaning → Groupby feature engineering (velocity, counts) → Scaling vectors → Isolation Forest inference → Risk Score normalization (0-100) → SAR Generation via RAG.

**11. Is your system Real-Time or Batch?**
**Answer:** Currently, it operates as a Batch Analytics Engine (processing historical CSV dumps). Real-time would require streaming architecture (e.g., Apache Kafka) to block transactions mid-flight.

**12. What was the most challenging part of this project?**
**Answer:** Preventing the Generative AI from "hallucinating" fake laws when writing SARs. I solved this by implementing a strict RAG architecture that forces the LLM to only cite PMLA clauses retrieved from a local FAISS vector database.

**13. What is the difference between KYC and AML?**
**Answer:** KYC (Know Your Customer) is onboarding identity verification. AML (Anti-Money Laundering) is the continuous behavioral monitoring of transactions post-onboarding.

**14. What are the three stages of Money Laundering?**
**Answer:** Placement (injecting dirty money into banks), Layering (moving it constantly to confuse the trail), and Integration (buying clean assets like real estate). My system focuses heavily on catching the Layering phase.

**15. How did you validate that your system works without real money laundering data?**
**Answer:** Because it is an unsupervised isolation model, "ground truth" labels aren't strictly required. I injected synthetic laundering topologies (fake round-trips and structurings) into normal transactional datasets and verified that those specific accounts scored in the high 90s.

---

## Part 2: Python, Pandas & Data Engineering (Q16 - Q40)
**16. What is Data Normalization in your pipeline?**
**Answer:** Transforming wildly inconsistent bank datasets into a standardized format. For example, dynamically mapping headers so that fields named `txn_value`, `transfer_amt`, or `amount` all unify into an internal `amount` column.

**17. Why use Pandas over standard Python `for` loops?**
**Answer:** Performance. Python `for` loops process data row-by-row in the interpreter, creating massive overhead. Pandas relies on "Vectorization"—pushing the math down to C-level arrays in contiguous memory. What takes 10 minutes in a Python loop takes 2 seconds in Pandas.

**18. Give an example of a Pandas vectorized operation you used.**
**Answer:** `df['is_high_value'] = df['amount'] > 50000`. This instantly evaluates an entire column simultaneously.

**19. How did you extract the Structuring frequency?**
**Answer:** `df[(df['amount'] > 40000) & (df['amount'] < 50000)].groupby('account_id').size()`. This vectorizes the filter and groups the counts instantly.

**20. What is a DataFrame?**
**Answer:** A 2-dimensional, mutable tabular data structure with labeled axes (rows and columns), essentially an in-memory SQL table.

**21. How do you handle missing (Null/NaN) values?**
**Answer:** Numerical columns (like amount) were filled with `0.0`. Categorical strings (like counterparty) were filled using `.fillna("UNKNOWN")`. Rows missing a primary key (`account_id`) were dropped completely via `.dropna()`.

**22. How did you convert dirty text like "₹50,000.00" into usable floats?**
**Answer:** I wrote a parser using string replacement/regex: `str(val).replace('₹', '').replace(',', '').strip()`, and safely applied `float()` mapping onto the column.

**23. What happens if Pandas encounters an unparseable date format?**
**Answer:** I used `pd.to_datetime(df['date'], errors='coerce')`. The `coerce` argument intercepts fatal parsing errors and quietly converts unparseable junk into `NaT` (Not a Time), allowing the surrounding pipeline to survive instead of crashing.

**24. What is the difference between `.merge()` and `.concat()`?**
**Answer:** `.concat()` stacks DataFrames (vertically adding new transaction rows). `.merge()` acts like a SQL JOIN, combining DataFrames horizontally based on a shared key (like merging a velocity DataFrame with a volume DataFrame on `account_id`).

**25. How do you handle massive datasets larger than server RAM?**
**Answer:** Pandas loads entirely into memory, which crashes the server. To solve this, you use the `chunksize=100_000` argument in `pd.read_csv`, processing the file piece by piece, or migrate from Pandas to Dask/PySpark which handles out-of-core computing natively.

**26. What Python data structures did you use outside of Pandas?**
**Answer:** Dictionaries for dynamic schema mapping (`{ 'amt': 'amount' }`), and Sets to uniquely track highly duplicated counterparty strings in O(1) time.

**27. What is exploratory data analysis (EDA)?**
**Answer:** Summarizing data characteristics before modeling. Checking min/max ranges, checking class imbalances, and plotting histograms to see that transaction values follow a heavy exponential decay distribution.

**28. How does `apply()` work in Pandas, and why should it be avoided if possible?**
**Answer:** `.apply()` runs a custom Python function row-by-row. It breaks vectorization, dropping performance back down to Python loop speeds. It should only be used as a last resort for complex regex or NLP string manipulation.

**29. What is a rolling window calculation?**
**Answer:** `df.rolling(window='7D').sum()`. It calculates a metric (like total transaction volume) for the specific 7 days preceding each exact row, necessary for temporal fraud typologies.

**30. How do you prevent thread locks during ingestion?**
**Answer:** By wrapping the ingestion parsing strictly within Python's `threading.Thread(target=process)`. This allows the UI to remain unblocked and instantly respond, pushing the massive analytic block into background space.

**31. What is an inner join vs an outer join in your Pandas merges?**
**Answer:** I mostly used Left Joins on the `account_id` base matrix, ensuring that every account remained in the final dataset even if they lacked data for a specific engineered anomaly feature.

**32. What is `scikit-learn` primarily used for in your pipeline?**
**Answer:** Mathematical preprocessing (`StandardScaler`, `MinMaxScaler`) and algorithm modeling (`IsolationForest`).

**33. How do you optimize high-throughput DB writes from a Pandas dataframe?**
**Answer:** Converting the final matrix into a list of model instances and passing them to an ORM bulk insertion method (or a multi-row raw SQL INSERT) in chunks of 500-1000, reducing remote network overhead drastically.

**34. Why do you need standard scaling (`StandardScaler`)?**
**Answer:** Because Isolation Forest and other distance trees calculates geometric splits. If "Transaction Count" ranges from 1-10 (small scale) but "Volume" ranges from 1-1,000,000 (large scale), the model completely ignores the Count. Standard Scaler forces all geometries to a mean of 0 and std-dev of 1.

**35. How is standard scaling calculated?**
**Answer:** For every value, subtract the mean of the column, and divide by the standard deviation: `z = (x - mean) / std`.

**36. Can you load non-CSV data (e.g., JSON or SQL dumps) into your pipeline?**
**Answer:** Yes. Pandas has native drivers like `pd.read_json()` and `pd.read_sql()`. The internal pipeline downstream requires zero changes because everything relies purely on the resultant standardized DataFrame structure.

**37. How would you handle continuous duplicate rows in the dataset?**
**Answer:** Using `df.drop_duplicates(subset=['account_id', 'transaction_id', 'amount'])` to guarantee we don't accidentally double-count a criminal's transaction volume due to a raw data export glitch.

**38. What is the Big-O complexity of looking up a dictionary key in Python?**
**Answer:** O(1) on average. This is why our schema normalization maps column names instantly, irrespective of dataset scale.

**39. How do you sort a DataFrame by anomaly score?**
**Answer:** `df.sort_values(by='risk_score', ascending=False)`. This is essential because compliance queues must be prioritized by highest risk first.

**40. What is "Feature Engineering"?**
**Answer:** The process of converting raw structured data (like a date, an amount, and an ID) into meaningful mathematical indicators (like "Velocity Ratio" or "Structuring Count") that explicitly expose patterns to a Machine Learning algorithm.

---

## Part 3: Python Architecture & OOP Concepts (Q41 - Q55)
**41. In your project, were OOP concepts used? If yes, how and where?**
**Answer:** Yes. The core analytical engine is structured as an `AnomalyDetector` Class. This demonstrates **Encapsulation** (hiding the ML scaler and model state inside the class), and the RAG layer is a `LegalRAGAgent` Class that encapsulates connecting to FAISS and building prompts, hiding those dependencies from the API runner.

**42. How did you use the `__init__` constructor?**
**Answer:** To initialize the class state. In the `AnomalyDetector`, `__init__` boots up an empty `IsolationForest(contamination=0.10)` model and an empty `StandardScaler()`, keeping them ready in memory to be `fit` or used for `predict`.

**43. Give an example of Encapsulation in your pipeline.**
**Answer:** The user calling the system only interfaces with a single method: `agent.generate_sar(data)`. They have zero visibility into the fact that the agent converts their data into an embedding, opens a FAISS database, retrieves 3 chunks of PMLA laws, formats a prompt, and streams a LangChain call. All of that complexity is protected inside the Class.

**44. Where is Polymorphism evident in Python?**
**Answer:** In the way we can write `len("string")` or `len([list])`. Or in our pipeline, `model.fit(df)` can accept a Pandas DataFrame or a raw NumPy array, polymorphically adapting to the data type provided.

**45. Did you use Static Methods (`@staticmethod`)?**
**Answer:** Yes, for pure functional, side-effect-free methods. For example, `clean_currency(text)` is a static method inside the class. It accepts a string and returns a float. It does not require access to `self` because it doesn't modify the state of the overall Anomaly Detector.

**46. Did you use Class Methods (`@classmethod`)?**
**Answer:** We primarily relied on standard instance methods for the ML lifecycle, but class methods are excellent for writing alternate constructors (e.g., `AnomalyDetector.from_pretrained_file("model.joblib")`).

**47. What is Inheritance, and how does it play into backend web frameworks?**
**Answer:** Inheritance allows a class to inherit behaviors from a parent. In almost all Python API frameworks, an endpoint handler inherits from a base `APIView` or `RequestHandler` class, meaning you instantly inherit HTTP parsing capabilities without writing boilerplate code.

**48. Why architecture your ML code as a Class instead of a global functions file?**
**Answer:** Because an ML pipeline holds mutable state. Over the course of processing, the `StandardScaler` calculates the dataset's standard deviation and expects to hold onto it in memory for future incoming data. Passing that matrix around globally via function arguments creates horrible, fragile spaghetti code. Storing it safely within `self.scaler` is clean.

**49. How do you handle Exceptions safely?**
**Answer:** By encapsulating risky operations in `try/except` blocks. If an incoming file lacks a mandatory `amount` column, we `raise ValueError("Missing critical amount header")` early and cleanly, returning an error payload rather than allowing Pandas to abruptly crash the runtime daemon stringing a cryptic traceback.

**50. What is a Python context manager (`with` statement)?**
**Answer:** It guarantees resource cleanup. We used it extensively when opening the FAISS vectors or temporary files: `with open('book.md', 'r') as file:` ensures that even if an exception breaks the code inside the block, the file handle is safely closed, preventing memory leaks.

**51. What is the difference between a list and a tuple in Python?**
**Answer:** Lists are mutable (can be changed). Tuples are immutable. I use tuples to pass strictly defining structures (like `(window_size, threshold)`) ensuring configuration constants cannot be accidentally altered by downstream code.

**52. How does the `yield` keyword work?**
**Answer:** It converts a function into a Generator. It returns data one piece at a time and suspends its state. Extremely useful for processing 50-million row data arrays without loading them entirely into memory.

**53. What is PEP 8?**
**Answer:** The style guide for Python code. Maintaining strict PEP 8 compliance (snake_case variables, CamelCase classes, 80-char wrap) is critical for enterprise software readability, specifically when dealing with dense mathematical ML matrices.

**54. What are Python Decorators?**
**Answer:** Functions that modify the behavior of other functions. We used decorators (like `@api_view` or `@staticmethod`) to dynamically inject behavior (like enforcing authentication or HTTP routing) without modifying the internal math of the function itself.

**55. How do you manage Python dependencies so they are portable?**
**Answer:** By maintaining a `requirements.txt` listing exact version tags (`pandas==2.0.3`). Failing to lock versions causes the pipeline to break when algorithms update internal math formulas on a server you deploy to.

---

## Part 4: Machine Learning — Isolation Forest & Scaling (Q56 - Q75)
**56. Why is Unsupervised Learning necessary for AML?**
**Answer:** Supervised ML requires highly labeled data (a truth column telling the model exactly rows which are fraud). Banks do not share this data, and fraud represents just 0.01% of all activity. We MUST use Unsupervised Learning, which mathematically maps what "normal" behavior looks like and flags any data point that sits completely outside that cluster.

**57. Exactly how does Isolation Forest detect an anomaly?**
**Answer:** It's an ensemble of decision trees. It randomly selects a feature and places a random split line. It continues splitting until all data points are isolated into their own terminal nodes. An anomaly (like a massive Structuring operation) requires very, very few splits to be isolated. A normal transaction is buried deep in a massive cluster and requires dozens of splits. 

**58. How does Path Length determine the score?**
**Answer:** Shorter average path length across all trees = Highly Anomalous. Longer path length = Normal. The algorithm aggregates these paths to produce a unified score between -1 and 1.

**59. What is the `contamination` hyperparameter?**
**Answer:** It acts as an expected baseline limit. Setting `contamination=0.10` tells the Isolation Forest that roughly 10% of the parsed matrix will be considered strictly anomalous behavior.

**60. What happens if contamination is set too high (e.g., 0.50)?**
**Answer:** You recreate the exact problem of legacy systems: massive False Positives. It forces the model to flag roughly 50% of the dataset as fraudulent, destroying the compliance team's review queue.

**61. What happens if contamination is too low (e.g., 0.0001)?**
**Answer:** You suffer extreme False Negatives. The model becomes too loose, and actual money laundering configurations are accepted as normal behavior. 

**62. How do you serialize an ML model to disk?**
**Answer:** Model training takes computational time. Once trained, I serialize the model artifact to binary files using `joblib.dump(model, "pipeline.joblib")`. This allows instant loading and scoring in the future without retraining everything.

**63. What is the Curse of Dimensionality?**
**Answer:** As you add more columns/features to a model, the mathematical hyperspace expands exponentially until all data points appear equally distant from each other, breaking distance-based algorithms. This is why I aggressively narrowed the engineered features down to just 4 distinct risk vectors.

**64. How did you convert the Isolation Forest raw output to a 0-100 Risk Score?**
**Answer:** The Isolation Forest predicts values where -1 is highly anomalous. I inverse the polarity (multiplying by -1), and pass it through a `MinMaxScaler(feature_range=(0,100))` to give analysts a standard percentage reading.

**65. Why not use K-Means Clustering?**
**Answer:** K-Means forces every data point into a specific cluster centroid. Outliers significantly drag and warp the centroids, making it highly sensitive and less accurate for anomaly isolation. Also, K-Means assumes clusters are perfectly spherical, which banking behavior never is.

**66. What is the difference between One-Class SVM and Isolation Forest?**
**Answer:** One-Class SVM attempts to draw a hard non-linear boundary around normal data. It scales terribly with large datasets (O(n³)). Isolation Forest builds isolated split trees, scaling incredibly efficiently (O(n log n)) and utilizing very low RAM.

**67. Is XGBoost suitable for your project?**
**Answer:** No. XGBoost is predominantly a Supervised algorithm requiring ground-truth labeled training targets. I don't possess a massive labeled dataset of true money launderers.

**68. If a new, highly complex typological fraud trend appears, will your model catch it?**
**Answer:** Yes, if the new fraud's fundamental mathematical geometry is completely separate from normal banking behavior. That's the beauty of unsupervised AI—it doesn't need to be trained on the new fraud; it just needs to recognize that the behavior is "not normal."

**69. How do you prevent features from overpowering each other?**
**Answer:** (Crucial) Standard Scaling. The `velocity` ratio might be `3.5`, but the `volume` might be `10,000,000`. Without `StandardScaler`, the 10-million value massively over-influences the math.

**70. What is Overfitting and Underfitting? Does it apply here?**
**Answer:** Overfitting is when a model memorizes the training data perfectly but fails to generalize. Underfitting is when the model is too simple. Because Isolation Forest is an unsupervised boundary estimator relying on randomized cuts, it natively resists heavy overfitting, though the contamination param acts as a regularizer.

**71. How do you evaluate an Unsupervised Model's performance without Answer Labels?**
**Answer:** In production environments, it is evaluated by Qualitative Hit Rate—sending the top 100 highest-scored anomalies to a human SME (Subject Matter Expert). If 80 of them are actually suspicious, the model is highly performant. If the SME rejects them all, the `contamination` boundary is too loose.

**72. Why didn't you feed the Account ID strings or Customer Names into the ML model?**
**Answer:** Algorithms require continuous mathematical numbers. Strings mean nothing. You could use One-Hot Encoding to convert names to 0/1 columns, but encoding 100,000 unique customer names would create 100,000 extra dummy columns—creating instant RAM explosion and Dimensionality Curse.

**73. What is Feature Importance, and can Isolation Forest provide it?**
**Answer:** Feature Importance ranks exactly which column drove the anomaly score. Vanilla Isolation Forest struggles with explainability. To provide this, we would introduce SHAP (SHapley Additive exPlanations) values around the backend to unpack the model reasoning.

**74. Did you tune any Hyperparameters besides `contamination`?**
**Answer:** Yes, the number of estimators (`n_estimators`). Utilizing 100 decision trees is the mathematical consensus for balancing detection edge cases against heavy computational processing overhead.

**75. How does the model perform if a customer exhibits two different typologies simultaneously?**
**Answer:** Structuring combined with Round-Tripping aggressively isolates the path length across multiple tree nodes exponentially faster, resulting in a devastating 99-tier Critical Risk Score.

---

## Part 5: RAG, Generative AI & Vectors (Q76 - Q85)
**76. Explain RAG (Retrieval-Augmented Generation) in 30 seconds.**
**Answer:** Large Language Models (LLMs) hallucinate facts they don't know. To use AI in a legal environment, we don't ask it to generate from memory. We query a database of verified legal laws, retrieve the exact law paragraph, hand it to the LLM, and explicitly prompt it: *"Write your report using strictly this text alone."* RAG grounds generative text in retrieved truth.

**77. What is an Embedding?**
**Answer:** A technology that takes text (a sentence of PMLA law) and converts it into a high-dimensional mathematical coordinate (usually an array of 768 or 1536 float numbers). This allows computers to understand semantic meaning and intent, not just raw keywords.

**78. What is a Vector Database, and why FAISS?**
**Answer:** Relational databases natively store rows; vector databases store Embeddings (math arrays). You query a vector database by searching for coordinates that are mathematically "close" (Cosine Similarity). FAISS (Facebook AI Similarity Search) is a blazing-fast library optimized for exactly this vector proximity lookup.

**79. What LLM did you use and why?**
**Answer:** **Llama-3.3-70b-versatile** accessed via the **Groq API**. I selected Groq because it uses LPUs (Language Processing Units) rather than traditional GPUs, allowing inference speeds exceeding 300 words a second, which provides a near-instant user experience for generating massive legal reports.

**80. What is LangChain?**
**Answer:** A Python orchestration framework. Instead of manually writing HTTP requests to the LLM API and explicitly pasting database retrieval text together, LangChain stitches the Vector DB search, the LLM memory, and the custom instructions into an elegant executable "chain."

**81. Why is setting `temperature=0.1` critical?**
**Answer:** For generating creative poetry, you want high temperature (1.0). For generating regulatory Suspicious Activity Reports (SARs) that will be read by government agencies, you absolutely cannot have AI creativity. Temperature 0.1 restricts the LLM to highly deterministic, highly factual structure parsing output.

**82. What happens if the RAG retrieves the wrong law?**
**Answer:** The LLM will confidently write a report citing the wrong law. This emphasizes why the vector Embedder model and the text "Chunking Strategy" (how large a piece of text to parse into FAISS) are heavily crucial in the backend.

**83. How do you prevent prompt injection?**
**Answer:** The system doesn't accept free-text inputs from users. The transaction data context is passed entirely programmatically from the ML pipeline directly into the system prompt securely.

**84. Why not just paste the entire PMLA Actbook into the LLM every time?**
**Answer:** LLMs have strict limits on how many words they can read at once (the "Context Window"), and API costs scale based on words inputted. RAG is cost-effective because it only sends the exact 3 paragraphs of relevant law instead of a 400-page book.

**85. Could your RAG system work for US laws like the Bank Secrecy Act?**
**Answer:** Perfectly. No code changes are required. You merely strip out the Indian PMLA PDF, ingest the US Bank Secrecy Act PDF into the FAISS vector database, and the system seamlessly pivots jurisdictions.

---

## Part 6: System Design, Scalability & Scenarios (Q86 - Q100)
**86. Scenario: The regulator audits your system and asks "Why was this specific alert generated?". How do you technically explain it?**
**Answer:** I point to the RAG-generated SAR report, which documents the exact Typology tags generated by the Rule-Based engine layer (e.g., "Structuring"). I show them that the Isolation Forest scored an isolation depth in the 95th percentile relative to the global data state.

**87. How does the architecture handle asynchronous workloads?**
**Answer:** Long-running analytic jobs are decoupled from API response HTTP handlers. I use Python background worker threads to trigger the pipeline, allowing the system to instantly return HTTP 202 ("Accepted") so the frontend can display a loading bar, instead of letting heavy math crash the web worker pool with timeouts.

**88. Scenario: An Analyst uploads a 100 Million row Excel file. What happens?**
**Answer:** In the current Pandas prototype, it causes an OOM (Out Of Memory) crash. To secure the architecture to enterprise level, I would rewrite the file read loop into a chunked iterator (processing 50,000 line batches sequentially) or substitute a distributed Dask cluster.

**89. Why is a File-based database like SQLite unacceptable for this specific project at scale?**
**Answer:** SQLite locks the entire database file upon writing. When 50 analysts attempt to bulk write massive ML analytic chunks simultaneously, violent concurrency locks and failures occur. Production requires scalable Postgres layer with dedicated connection pools.

**90. How do you handle writing 50,000 generated risk alerts efficiently?**
**Answer:** If you loop over objects and call `.insert()`, it issues 50,000 separate TCP/IP commands to the database engine. Instead, I append them into a massive array and use batch `bulk_insertions` set to a limit of 1000 items, drastically compressing network I/O blockings.

**91. What is continuous deployment (CI/CD)? Could it apply here?**
**Answer:** Yes. Implementing a GitHub Actions pipeline where pushing a code change triggers automated unit tests verifying the mathematical behavior of the Pandas structs, and builds a fresh Docker image of the entire backend.

**92. How would you handle a memory leak in the RAG execution?**
**Answer:** Langchain holds heavily onto prompt memory histories if configured wrong. I ensure explicit teardown of FAISS indices and vector representations by isolating them inside functional namespaces that garbage collect correctly.

**93. Scenario: The Machine Learning model's alerts degrade in accuracy after 6 months. Why?**
**Answer:** "Data Drift." Criminals adapt to the fact that their previous structuring models were caught, so they invent entirely new geometries. The unsupervised model must be periodically retrained (e.g., bi-monthly) to capture and isolate the new behavioral distributions.

**94. What is Big-O complexity of Isolation Forest Scoring vs Training?**
**Answer:** Training consists of building random binary trees, typically around O(N log N). Predict/Inference consists merely of tracking a value down the pre-built nodes, which scales efficiently in rapid `O(log N)`.

**95. How would you implement Graph Analytics to upgrade the system?**
**Answer:** Currently, we view round-trips solely as (Node A → Node B → Node A). Adding a Graph Database (like Neo4j or NetworkX) permits discovering 6-hop money trails where criminal networks cascade through layered offshore shell entities before returning.

**96. A transaction fails your data cleaning layer completely resulting in a fatal Python parsing crash. How do you prevent the whole pipeline dying?**
**Answer:** Robust `try/except` wrapper. I log the failing row signature to a `quarantine_logs` output and `.continue()` the sequence, ensuring that 99.9% of the dataset succeeds rather than failing fully for one broken datum index.

**97. What is Model Interpretability?**
**Answer:** The concept of mathematically proving why a black-box AI made a decision, typically using advanced mechanics like SHAP values to chart exactly how heavily a specific Pandas vector swayed the final Isolation Forest cut logic.

**98. How is Authentication / Authorization implemented on API endpoints conceptually?**
**Answer:** Endpoints must be secured behind JWT (JSON Web Tokens) or standard API Key Middleware structures. Unauthenticated users firing massive data processing payloads would DDOS the computation resources.

**99. Why not just use OpenAI's ChatGPT directly?**
**Answer:** Sending highly sensitive, classified global banking customer data to an open public API like OpenAI breaches every financial data privacy law on Earth (GDPR, DPDP). Systems must utilize private local LLMs or heavily ring-fenced enterprise secure API deployments (like Groq via strict agreements).

**100. Sum up the entire architectural achievement of your project in one sentence.**
**Answer:** My system proves that applying complex mathematical clustering uncovers hidden criminal behavior better than boolean logic, while demonstrating that applying autonomous RAG technology drastically modernizes the manual friction of the regulatory reporting space.

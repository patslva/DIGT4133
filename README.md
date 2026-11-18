# NorthLine Commerce: delivery-exception prediction (DIGT 4133 midterm)

Predicts, at the moment an order is dispatched, whether it will have a delivery exception within 7 days. PySpark workflow comparing a logistic-regression baseline with a tuned random forest, using only dispatch-time information.

## Files

| File | What it is |
|---|---|
| `NorthLine_Midterm_v2.ipynb` | The notebook: audit, cleaning, features, tuning, evaluation and plots (already run, outputs saved). |
| `NorthLine_Technical_Report_v2.pdf` | The technical report. |
| `Dataset.csv` | The supplied order data (the notebook reads it from its own folder). |
| `requirements.txt` | Python packages for the virtual environment. |

## 1. Install the prerequisites

You need **Python 3.12 or newer** and **Java 17 or 21** (Spark 4.0 supports 17 and 21; the reported results used OpenJDK 21).

**macOS (Homebrew):**
```bash
brew install python@3.13 openjdk@21
# make Java visible to Spark (add these two lines to ~/.zshrc to keep them)
export JAVA_HOME="$(brew --prefix openjdk@21)/libexec/openjdk.jdk/Contents/Home"
export PATH="$JAVA_HOME/bin:$PATH"
java -version        # should print version 21
```

**Windows / Linux:** install Python 3.12+ from python.org (or your package manager) and a JDK 17 or 21 (for example Temurin), then check `java -version`.

## 2. Create the Python virtual environment

From this repository's folder:

```bash
git clone https://github.com/patslva/DIGT4133.git
cd DIGT4133

python3 -m venv .venv
source .venv/bin/activate            # Windows: .venv\Scripts\activate
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m ipykernel install --user --name digt4133 --display-name "Python (DIGT4133 .venv)"
```

Check the install:

```bash
python -c "import pyspark, pandas, sklearn; print(pyspark.__version__, pandas.__version__, sklearn.__version__)"
# expected: 4.0.1 2.2.3 1.6.1
```

## 3. Run the notebook

```bash
jupyter lab NorthLine_Midterm_v2.ipynb
```

In JupyterLab choose the kernel **Python (DIGT4133 .venv)** (Kernel > Change Kernel), then **Run > Run All Cells**.

- `Dataset.csv` must be in the same folder as the notebook; no path needs editing.
- The full run takes about **27 minutes on 2 CPU cores** (most of it is the 36-forest tuning grid); it is faster on more cores.
- Quick smoke test (about 4 minutes, tiny search grids, results will differ from the report): set `NL_QUICK=1` in the terminal before starting Jupyter, for example `NL_QUICK=1 jupyter lab`.
- The notebook writes its tables and figures to an `outputs/` folder next to it, including `data_quality_log.csv` and `tuning_results.csv`.

## Reproducibility

Seed 4133 everywhere. Spark partitioning and row order are fixed (adaptive execution off, 8 shuffle partitions, rows sorted within partitions), so the random forest gives repeatable results. Test labels are only used in the final-evaluation section; every choice (model settings, alert cutoffs) is made on the validation split.

Versions used for the reported results: Python 3.13, PySpark 4.0.1, pandas 2.2.3, NumPy 2.2.2, scikit-learn 1.6.1, Matplotlib 3.10.1, OpenJDK 21.

## Troubleshooting

- `JAVA_HOME is not set` or `Unable to locate a Java Runtime`: install a JDK (step 1) and export `JAVA_HOME` in the same terminal that starts Jupyter.
- Spark fails with a Java version error: use Java 17 or 21, not 8, 11 or newer than 21.
- `FileNotFoundError: Put Dataset.csv next to the notebook`: start Jupyter from the folder that holds `Dataset.csv`.
- Out-of-memory in the tuning cell: close other apps, or lower `spark.driver.memory` in the first code cell if your machine has under 8 GB.

### Tools and Reproduction Steps

Follow these steps to replicate and verify the results presented in this analysis.

#### Prerequisites

- **Docker** installed and running
- **Git** installed
- **Python** (v3.11 used)

#### Step-by-Step Instructions

1. **Clone the Repository:**

```bash
git clone https://github.com/miri1337/imago-data-challenge.git
cd imago-data-challenge
```

2. **Start SQL Server in Docker:**

```bash
docker-compose up -d
```

Check if SQL Server is running:

```bash
docker ps
```

3. **Setup Python Environment:**

```bash
python -m venv venv
source venv/bin/activate  # Linux/macOS
venv\Scripts\activate     # Windows

pip install -r requirements.txt
```

4. **Data Import to SQL Server:**

Run the Jupyter Notebook responsible for importing CSV data into the database:

```bash
jupyter notebook notebooks/analysis.ipynb
```




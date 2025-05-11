# Jules Backlog Demo
## Context
This repo is a fork of the Pandas library and is intended to show off how Jules can tackle multiple bugs on a backlog at once. 

Jules documentation can be found at [x].

## Overview

## Environment Set Up Scripts
To set up the correct dev environment, you can use this bash script in Jules:

```
#!/bin/bash

# Exit immediately if a command exits with a non-zero status.
set -e

# --- Configuration ---
# Adjust this if your Python 3 version is different (e.g., 3.10, 3.11)
# The script will try to install pythonX.Y-venv and pythonX.Y-dev
PYTHON_VERSION_MAJOR_MINOR="3.12" # As seen in your error log

# Path to your pandas source code directory
PANDAS_SRC_DIR="/app"

# Path where the virtual environment will be created
VENV_DIR="/home/swebot/virtualenvs/pandas-dev"
# --- End Configuration ---

echo "--- Starting pandas development environment setup ---"

# 1. Update package lists
echo ">>> Updating package lists..."
sudo apt update || true

# 2. Install python3-venv package
echo ">>> Installing python${PYTHON_VERSION_MAJOR_MINOR}-venv..."
sudo apt install -y "python${PYTHON_VERSION_MAJOR_MINOR}-venv"

# 3. Install python3-dev package (FOR PYTHON.H AND OTHER DEV HEADERS)
echo ">>> Installing python${PYTHON_VERSION_MAJOR_MINOR}-dev..."
sudo apt install -y "python${PYTHON_VERSION_MAJOR_MINOR}-dev" # 

# 4. Navigate to the pandas source directory
echo ">>> Changing to pandas source directory: ${PANDAS_SRC_DIR}"
cd "${PANDAS_SRC_DIR}"

# 5. Create the virtual environment
echo ">>> Creating Python virtual environment at: ${VENV_DIR}"
# Ensure the parent directory for venvs exists if not using ~
# mkdir -p "$(dirname "${VENV_DIR}")" # Uncomment if /home/swebot/virtualenvs might not exist
python3 -m venv "${VENV_DIR}"

# 6. Activate the virtual environment (for the context of this script)
echo ">>> Activating virtual environment..."
source "${VENV_DIR}/bin/activate"

# Check if activation was successful (pip should now point to venv)
echo ">>> Pip path after activation: $(which pip)"
echo ">>> Python path after activation: $(which python)"

# 7. Install build dependencies from requirements-dev.txt
echo ">>> Installing build dependencies from requirements-dev.txt..."
python -m pip install -r requirements-dev.txt

# 8. Fetch latest tags for pandas versioning
echo ">>> Fetching latest git tags for pandas..."
if git remote -v | grep -q upstream; then
    echo "Upstream remote already exists."
else
    echo "Adding upstream remote: https://github.com/pandas-dev/pandas.git"
    git remote add upstream https://github.com/pandas-dev/pandas.git
fi
git fetch upstream --tags

# 9. Build and install pandas in editable mode using meson
echo ">>> Building and installing pandas (editable mode with verbose output)..."
python -m pip install -ve . --no-build-isolation -Ceditable-verbose=true

echo "--- Pandas development environment setup complete! ---"
echo "To activate the environment in your current terminal session manually, run:"
echo "source ${VENV_DIR}/bin/activate"
echo "Once activated, you should be able to 'import pandas' and see your dev version."
```

## Prompts

| Prompt # | Bug Report                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                    |
| -------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1        | Pandas version checks<br><br>I have checked that this issue has not already been reported.<br><br><br>I have confirmed this bug exists on the latest version of pandas.<br><br><br>I have confirmed this bug exists on the main branch of pandas.<br><br>Reproducible Example<br>import pandas as pd<br><br>\# Single timestamp<br>raw = "2023-10-15T14:30:00"<br>single = pd.to_datetime(raw, utc=True, format="ISO8601")<br>print(single)<br>\# Output: 2023-10-15 14:30:00+00:00 (correct)<br><br>\# Series of timestamps<br>series = pd.Series([0, 0], index=["2023-10-15T10:30:00-12:00", raw])<br>converted = pd.to_datetime(series.index, utc=True, format="ISO8601")<br>print(converted)<br>\# Output: 2023-10-16 02:30:00+00:00 for the second one (incorrect)<br>\# error depends on the previous one timezone<br>Issue Description<br>When using pd.to_datetime to parse a Series of timestamps with format="ISO8601" and utc=True, the parsing of a timestamp without an explicit timezone offset is incorrect and appears to depend on the timezone offset of the previous timestamp in the Series. This behavior does not occur when parsing a single timestamp.<br><br>Expected Behavior<br>In this configuration, behavior should not depend on the previous timestamp timezone. Result should be the same as when individually passed.<br><br>When you batch‐parse ["2023-10-15T10:30:00-12:00", "2023-10-15T14:30:00"] with format="ISO8601", utc=True, the second (naive) timestamp wrongly reuses the “–12:00” offset and becomes 2023-10-16T02:30:00+00:00 instead of 2023-10-15T14:30:00+00:00. The ISO8601 parser is retaining its last‐seen offset between parses.<br><br>I believe when utc=True, naive timestamps should be treated as UTC.                                                                                                                                                                                                                                                                                                         |
| 2        | Pandas version checks<br><br>I have checked that this issue has not already been reported.<br><br><br>I have confirmed this bug exists on the latest version of pandas.<br><br><br>I have confirmed this bug exists on the main branch of pandas.<br><br>Reproducible Example<br>import pandas as pd<br><br>series_result = pd.Series({"a": 1.0}, dtype="Float64").dot(<br>pd.DataFrame({"col1": {"a": 2.0}, "col2": {"a": 3.0}}, dtype="Float64")<br>)<br>series_result.dtype # is dtype('O')<br><br>series_result_2 = pd.Series({"a": 1.0}, dtype="float[pyarrow]").dot(<br>pd.DataFrame({"col1": {"a": 2.0}, "col2": {"a": 3.0}}, dtype="float[pyarrow]")<br>)<br>series_result_2.dtype # same, is dtype('O')<br><br>\# \`DataFrame.dot\` was already fixed<br>df_result = pd.DataFrame({"col1": {"a": 2.0}, "col2": {"a": 3.0}}, dtype="Float64").T.dot(<br>pd.Series({"a": 1.0}, dtype="Float64")<br>)<br>df_result.dtype # is Float64Dtype()<br>Issue Description<br>Series.dot with Arrow or nullable dtypes returns series result with numpy object dtype. This was reported in #53979 and fixed for DataFrames in #54025.<br><br>Possibly side notes: I believe the "real" issue here is that the implementation uses .values which returns a dtype=object array for the DataFrame. This seems directly related to #60038 and at least somewhat related to #60301 (which is also referenced in a comment on the former).<br><br>Expected Behavior<br>I would expect Series.dot to return the "best" common datatype for the input datatypes (in the examples, would expect the appropriate float dtype)<br><br>import pandas as pd<br><br>series_result = pd.Series({"a": 1.0}, dtype="Float64").dot(<br>pd.DataFrame({"col1": {"a": 2.0}, "col2": {"a": 3.0}}, dtype="Float64")<br>)<br>series_result.dtype # would expect Float64Dtype()<br><br>series_result_2 = pd.Series({"a": 1.0}, dtype="float[pyarrow]").dot(<br>pd.DataFrame({"col1": {"a": 2.0}, "col2": {"a": 3.0}}, dtype="float[pyarrow]")<br>)<br>series_result_2.dtype # would expect float[pyarrow] |
| 3        | Pandas version checks<br><br>I have checked that this issue has not already been reported.<br><br><br>I have confirmed this bug exists on the latest version of pandas.<br><br><br>I have confirmed this bug exists on the main branch of pandas.<br><br>Reproducible Example<br>import pandas as pd<br><br>x = pd.Series([1,2,3,5])<br>y = pd.Series([2,3,4])<br><br>pd.eval("(x + y).dropna()")<br>\# raises AttributeError: 'BinOp' object has no attribute 'value'<br><br>\# Note that something like pd.eval("(x.dropna() + y)") works!<br>Issue Description<br>Also effects other Series methods called on the result of a binary operation, which work well if applied to one operand.<br><br>See also #24670<br><br>Expected Behavior<br>Should return a Series of sums with nan values removed. Generally I would expect that it should be possible to apply a Series method to the result of a binary operation as part of an eval expression.<br><br>Installed Versions                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                            |
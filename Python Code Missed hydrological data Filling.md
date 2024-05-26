# Import the necessary libraries
import pandas as pd
from fancyimpute import IterativeImputer
import numpy as np
import matplotlib.pyplot as plt
from sklearn.experimental import enable_iterative_imputer
from sklearn.impute import IterativeImputer

# Load data
data = pd.read_csv(r"C:\Users\user\Desktop\Data\Data Analysis\MilleHom.csv")
df=pd.DataFrame(data)
print(df)

# Assuming 'data' is your DataFrame
# Check for missing values before imputation
print(data.isnull().sum().sum())

# Define the strings representing missing values and their replacements
missing_replacements = {
    ' ': np.nan,
    '-   ': np.nan,
    '-': np.nan,
    '         ': np.nan  # Adjust this string according to the specific representation in your data
}

# Define the value representing 0
zero_replacement = 0

# Replace 0 values with NaN
data.replace(zero_replacement, np.nan, inplace=True)

# Replace string values with NaN
for missing_string, replacement in missing_replacements.items():
    data.replace(missing_string, replacement, inplace=True)

# Fill missing data using the Multiple inputation method
imputer = IterativeImputer(max_iter=10, random_state=0)
filled_data = imputer.fit_transform(data)


# Convert the imputed array back to a DataFrame
filled_data_df = pd.DataFrame(filled_data, columns=data.columns)

# Check for missing values after imputation
print(filled_data_df.isnull().sum().sum())

# Print the results
print(filled_data_df)

filled_data_df

filled_data_df.to_csv(r'C:\Users\user\Desktop\Data\Data Analysis\From WoWE\Logia_WL1.csv', index=False)










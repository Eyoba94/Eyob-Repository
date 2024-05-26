# Import necessary Libraries
import numpy as np
import pandas as pd
import pyhomogeneity as hg
import matplotlib.pyplot as plt
import statsmodels.api as sm
%matplotlib inline

# read all datasets
data = pd.read_csv(r"C:\Users\user\Desktop\Data\Data Analysis\Homogenization\Logia_Hom6.csv")

data.index = pd.to_datetime(data['Year'], format='%Y-%m-%d')
data.drop('Year', axis=1, inplace=True)

# Now your data should be indexed by the year
print(data.head())
data

# Plotting
plt.figure(figsize=(16, 6))
data.plot(figsize=(16, 6))
plt.xlabel('Year')
plt.ylabel('Flow')
plt.grid(True)
plt.show()

# Performs the Pettitt test on the 'data' to detect a change point
pettitt_res = hg.pettitt_test(data, alpha=0.05)
pettitt_res
# Performs the Buishand U test on the 'data' to detect a shift in the mean
buishand_res = hg.buishand_u_test(data)
buishand_res

# Extracting results and data information
result = pettitt_res
mn = data.index[0]
mx = data.index[-1]
loc = pd.to_datetime(result.cp)
mu1 = result.avg.mu1
mu2 = result.avg.mu2

# Setting up plot parameters
plt.figure(figsize=(15,10))
plt.plot(data, label="Observation")
plt.hlines(mu1, xmin=mn, xmax=loc, linestyles='--', colors='orange', lw=1.5, label='mu1 : ' + str(round(mu1, 2)))
plt.hlines(mu2, xmin=loc, xmax=mx, linestyles='--', colors='g', lw=1.5, label='mu2 : ' + str(round(mu2, 2)))
plt.axvline(x=loc, linestyle='-.', color='red', lw=1.5, label='Change point : ' + loc.strftime('%Y-%m-%d') + '\n p-value : ' + str(result.p))

# Customizing plot appearance
plt.title('')
plt.xlabel('Year', fontweight='bold', fontsize=12)
plt.ylabel('Flow', fontweight='bold', fontsize=12)

# Enhancing axis and legend appearance
plt.xticks(fontweight='bold', fontsize=10)
plt.yticks(fontweight='bold', fontsize=10)

legend = plt.legend(loc='upper right')
for text in legend.get_texts():
    text.set_fontweight('bold')
    text.set_fontsize(10)

# Final plot adjustments and display
plt.grid(True)
plt.show()
# Now to detect change points and treat the data we can use the following codes.
# Step 1: Identify the change point detected by the test
change_point_index = data.index.get_loc(loc)

# Step 2: Split the dataset into two parts based on the change point
data_before_change = data.iloc[:change_point_index]
data_after_change = data.iloc[change_point_index:].copy()  # Make a copy to avoid modifying the original data

# Step 3: Apply some transformation to homogenize the data starting from the change point
# Here, you can apply any transformation method you prefer to homogenize the data
# For example, you can apply a simple mean adjustment to make the two segments continuous
mean_adjustment = mu2 - mu1
data_after_change -= mean_adjustment

# Step 4: Adjust the data after the change point to ensure non-negative values
data_after_change[data_after_change < 0] = 0

# Step 5: Concatenate the original data before the change point with the adjusted data after the change point
homogenized_data = pd.concat([data_before_change, data_after_change])

# Step 6: Save the modified dataset to a CSV file
homogenized_data.to_csv(r"C:\Users\user\Desktop\Data\Data Analysis\Homogenization\Logia_Hom6.csv")


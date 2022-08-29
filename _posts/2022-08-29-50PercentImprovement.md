---
layout: post
title: "What to Include in the Outcome Metrics for Depression Treatment?--Taking a Look at the 50% Improvements in Different Aspects of Depression"
categories: Descriptive, OutcomeMetrics
excerpt: When discussing treatment outcomes, we tend to focus solely on symptom reduction. We typically assume that other aspects of improvements always follow core symptom reduction. However, researchers rarely examine the overlaps between core symptom reduction and different types of outcome improvements. Here, we examined whether other types of outcome improvements follow core symptom reduction (here, depressive symptoms) in patients with depression after completing a partial hospital program. 
---

When discussing treatment outcomes, we tend to focus solely on symptom reduction. We typically assume that other aspects of improvements always follow core symptom reduction. However, researchers rarely examine the overlaps between core symptom reduction and different types of outcome improvements. 

Here, we examined whether other types of outcome improvements follow core symptom reduction (here, depressive symptoms) in patients with depression after completing a partial hospital program.  

We define clinically significant improvement as 50% of improvement in one measured domain. 

We examined the overlaps of individuals with  50% improvement in reduction of depressive symptoms with 50% of improvements in:
<ol>
    <li>Reduction of non-depressive symptoms</li>
    <li>Positive mental health</li>
    <li>Functioning</li>
    <li>Well-being</li>
</ol>

We also compared the global patient rating of overall improvement in each response group: 
<ol>
    <li>Responders to both domains (i.e., reduction of depressive symptoms and another measured domain)</li>
    <li>Responders to depressive symptom reduction only</li>
    <li>Responders to the other domain only</li>
    <li>Non-responders to both domains</li> 
</ol>


### Load Packages


```python
import pandas as pd
from dfply import *
import numpy as np
from plotnine import *
import statsmodels.api as sm
from statsmodels.formula.api import ols
from scipy.stats import ttest_ind
```

### Customize Notebook Displays


```python
pd.options.display.max_seq_items=300
pd.options.mode.chained_assignment = None  # default='warn'
warnings.simplefilter(action='ignore', category=FutureWarning)
warnings.simplefilter(action="ignore", category=SettingWithCopyWarning)
```

### Load Datasets


```python
#RDQ: Different Types of Treatment Outcomes
rdqPre = pd.read_spss('../Data/PrePostMeasures/RDQPre_1.sav') #Pre-treatment
rdqPost = pd.read_spss('../Data/PrePostMeasures/RDQPost_1.sav') #Post-treatment

#Discharge Package: Patient perceived satisfaction & Improvement 
satOld = pd.read_spss('../Data/PHPSatisfaction/EndofTreatmentSurveyData/EOT Verfication_1.sav')#Old data before 2020
satNew = pd.read_spss('../Data/DischargePacket/2022-06-06 dc.sav') #New data after 2020
#We swtiched the platform for data collection in 2020 so we have two separate datasets

#Diagnosis
dx = pd.read_spss('../Data/DemosDx/Diagnosis_1.sav') 

#Demographics
demo = pd.read_spss('../Data/DemosDx/Demographics Form_1.sav')
```

### Preprocess Data


```python
#Change ID type from float to object
rdqPre.ID1 = rdqPre.ID1.astype('object') 
rdqPost.ID1 = rdqPost.ID1.astype('object') 

#Replace text with level orders 
rdqPre = rdqPre.replace(
    {'not at all or rarely true': 0, 'sometimes true': 1, 'often or almost always true': 2})
rdqPost = rdqPost.replace(
    {'not at all or rarely true': 0, 'sometimes true': 1, 'often or almost always true': 2})

#Convert reverse items in RDQ
for i in [29,30,48,49,50,51,52] :
    colname1 = 'rdqpre_' + str(i) + '_1'
    colname2 = 'rdqpost_' + str(i) + '_1'
    rdqPre[colname1] = abs(rdqPre[colname1].astype('float') - 2)
    rdqPost[colname2] = abs(rdqPost[colname2].astype('float') - 2)
    
#Rename variables to ensure consistency between the old and new datasets of the global rating of improvement
satGlobalOld = satOld.loc[:,['PHP_ID_1', 'IMPRV_1']].rename(columns = {'PHP_ID_1': 'ID1','IMPRV_1': 'imprv1'})
satGlobalNew = satNew.loc[:,['id1', 'imprv_1']].rename(columns = {'id1': 'ID1','imprv_1': 'imprv1'})

#Merge old and new discharge package (i.e., overall satisfaction/improvement) datasets
imprv = pd.concat([satGlobalOld, satGlobalNew],axis = 0) #Use concact to merge data by columns
imprv = satGlobal[satGlobal.ID1 != 0] #Remove invalid IDs
imprv.ID1 = imprv.ID1.astype('object') #Change ID type from float to object
```


Create the five domains of improvements:
<ol>
    <li>Reduction in depressive symptoms</li>
    <li>Reduction in non-depressive symptoms</li>
    <li>Positive mental health</li>
    <li>Functioning</li>
    <li>Well-being</li>
</ol>

```python
rdqPre['pre_dsym'] = rdqPre.iloc[:,1:15].mean(axis = 1) #Pre-treatment Symptom Reduction in Depressive Symptoms
rdqPre['pre_ndsym'] = rdqPre.iloc[:,15:26].mean(axis = 1) #Pre-treatment Symptom Reduction in Non-Depressive Symptoms
rdqPre['pre_cope'] = rdqPre.iloc[:,26:31].mean(axis = 1) #Pre-treatment Coping
rdqPre['pre_pmh'] = rdqPre.iloc[:,31:43].mean(axis = 1) #Pre-treatment Positive Mental Health
rdqPre['pre_fun'] = rdqPre.iloc[:,43:53].mean(axis = 1) #Pre-treatment Functioning
rdqPre['pre_well'] = rdqPre.iloc[:,53:61].mean(axis = 1) #Pre-treatment Well-being

rdqPost['post_dsym'] = rdqPost.iloc[:,1:15].mean(axis = 1) #Post-treatment Symptom Reduction in Depressive Symptoms
rdqPost['post_ndsym'] = rdqPost.iloc[:,15:26].mean(axis = 1) #Post-treatment Symptom Reduction in Non-Depressive Symptoms
rdqPost['post_cope'] = rdqPost.iloc[:,26:31].mean(axis = 1)#Post-treatment Coping
rdqPost['post_pmh'] = rdqPost.iloc[:,31:43].mean(axis = 1) #Post-treatment Positive Mental Health
rdqPost['post_fun'] = rdqPost.iloc[:,43:53].mean(axis = 1) #Post-treatment Functioning
rdqPost['post_well'] = rdqPost.iloc[:,53:61].mean(axis = 1)#Post-treatment Well-being
```

Here we selected two groups of samples. 

One group contains all patients with a diagnosis of major depressive disorder (MDD). 
In contrast, the other group comprises only patients with MDD as their <i> primary </i> diagnosis (i.e., they primarily came here for depression).


```python
#Find patients with MDD
def findMDDAll(df: pd.DataFrame) -> pd.DataFrame:
    return df[df.mddsnp_1.str.contains('Curr|Prin') | df.mddrnp_1.str.contains('Curr|Prin')|
        df.mddsp_1.str.contains('Curr|Prin')| df.mddrp_1.str.contains('Curr|Prin')]

#Find patients with MDD as their primary diagnosis
def findMDDCurrPrin(df: pd.DataFrame) -> pd.DataFrame:
    return df[df.mddsnp_1.str.contains('Curr, Prin') | df.mddrnp_1.str.contains('Curr, Prin')|
        df.mddsp_1.str.contains('Curr, Prin')| df.mddrp_1.str.contains('Curr, Prin')]
    
```

### Compute Sample Sizes

Compute the sample sizes of MDD patients and primary MDD patients who completed:
<ol>
    <li>Pre-treatment assessments of different outcome domains</li>
    <li>Post-treatment assessment of different outcome domains</li>
    <li>Global patient ratings of overall improvements </li>
    <li>All the above three measurements</li>
</ol>

```python
#Complete Data of Pre-treatment RDQ
rdqPreComp = rdqPre.filter(regex = 'ID1|rdqpre_|pre_')
rdqPreComp = rdqPreComp.dropna() #Drop rows containing any missing values
rdqPreComp_dx = pd.merge(rdqPreComp, dx, on = 'ID1', how = 'inner') #Merge pre-treatment RDQ data with dianogses 

rdqPreCompMDDAll = findMDDAll(rdqPreComp_dx) #Find patients with MDD in the complete smaple
rdqPreCompMDDCurrPrin = findMDDCurrPrin(rdqPreComp_dx) #Find patients with primary MDD in the complete smaple


#Complete Data of Post-treatment RDQ
rdqPostComp = rdqPost.filter(regex = 'ID1|rdqpost|post_')
rdqPostComp = rdqPostComp.dropna() #Drop rows containing any missing values
rdqPostComp_dx = pd.merge(rdqPostComp, dx, on = 'ID1', how = 'inner') #Merge post-treatment RDQ data with dianogses 

rdqPostCompMDDAll = findMDDAll(rdqPostComp_dx) #Find patients with MDD in the complete smaple
rdqPostCompMDDCurrPrin = findMDDCurrPrin(rdqPostComp_dx) #Find patients with primary MDD in the complete smaple


#Complete Data of the Global Patient Rating of Improvement
imprvComp = imprv.dropna() #Drop rows containing any missing values
imprvComp_dx = pd.merge(imprvComp, dx, on = 'ID1', how = 'inner') #Merge post-treatment RDQ data with dianogses 

imprvCompMDDAll = findMDDAll(imprvComp_dx) #Find patients with MDD in the complete smaple
imprvCompMDDCurrPrin = findMDDCurrPrin(imprvComp_dx) #Find patients with primary MDD in the complete smaple

#Complete Data of the Above Assessments
rdqImprvDxComp = rdqPreComp.merge(rdqPostComp, on = 'ID1', how = 'inner') #Inner merging pre- and post-treatment RDQ
rdqImprvDxComp  = rdqImprvDxComp.merge(imprvComp, on = 'ID1', how = 'inner') #Inner merging Improvement
rdqImprvDxComp  = rdqImprvDxComp.merge(dx, on = 'ID1', how = 'inner') #Inner merging diagnoses

rdqImprvDxCompMDDAll = findMDDAll(rdqImprvDxComp)  #Find patients with MDD in the complete smaple
rdqImprvDxCompMDDCurrPrin = findMDDCurrPrin(rdqImprvDxComp) #Find patients with primary MDD in the complete smaple

sampleSize = pd.DataFrame([[rdqPreCompMDDAll.shape[0], rdqPreCompMDDCurrPrin.shape[0]],
                           [rdqPostCompMDDAll.shape[0], rdqPostCompMDDCurrPrin.shape[0]],
                           [imprvCompMDDAll.shape[0], imprvCompMDDCurrPrin.shape[0]],
                           [rdqImprvDxCompMDDAll.shape[0], rdqImprvDxCompMDDCurrPrin.shape[0]]],
                          columns = ['MDD ALL', 'PRINCIPLE MDD'],
                          index = ['PRE RDQ', 'POST RDQ', 'IMPROVEMENT', 'ALL ABOVE'])

sampleSize
```




<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>MDD ALL</th>
      <th>PRINCIPLE MDD</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>PRE RDQ</th>
      <td>2626</td>
      <td>1984</td>
    </tr>
    <tr>
      <th>POST RDQ</th>
      <td>1926</td>
      <td>1441</td>
    </tr>
    <tr>
      <th>IMPROVEMENT</th>
      <td>2118</td>
      <td>1558</td>
    </tr>
    <tr>
      <th>ALL ABOVE</th>
      <td>1126</td>
      <td>844</td>
    </tr>
  </tbody>
</table>
</div>



### Label Clinically Significant Improvement: 50% Improvement


```python
for sample in ['MDDAll', 'MDDCurrPrin']:
    dfMDD = locals().get('rdqImprvDxComp'+sample)

    labels = ['dsym', 'ndsym', 'cope', 'pmh', 'fun', 'well']

    #Compute Change Scores (Post - Pre RDQ)
    for label in labels:
        dfMDD['change_'+ label] = dfMDD.filter(regex='^post_'+label).squeeze() - dfMDD.filter(regex='^pre_'+label).squeeze()

    #Reverse symptom reduction scores (high scores = better improvement)
    dfMDD.change_dsym = dfMDD.change_dsym*(-1) 
    dfMDD.change_ndsym = dfMDD.change_ndsym*(-1)


    cut = 0.5 #Set a cutoff point
    for label in labels:
        dfMDD['change_'+label+'_cut'] = dfMDD['change_'+label] >= cut
        dfMDD['change_'+label+'_cut'] = dfMDD['change_'+label+'_cut'].replace({True: 1, False: 0}).astype('category')

    dfMDD.to_csv('../Data/dfMDD' + sample + '_satisfaction_rdq_dx_demo.csv') #Save processed data for later use
    
    

```

### Compute the Mean of the Patient Global Rating of Improvement in Each Responder Groups 


```python
for sample in ['MDDAll', 'MDDCurrPrin']:
    dfMDD = locals().get('rdqImprvDxComp'+sample)

    #Create a contingency table
    contingencyTable = pd.crosstab(dfMDD['change_dsym_cut'], dfMDD['change_'+'ndsym'+'_cut'])

    for label in labels[2:]:
        contingencyTable = pd.concat([contingencyTable, pd.crosstab(dfMDD['change_dsym_cut'], 
                                                                    dfMDD['change_'+label+'_cut'])], axis = 1)
        percentageTable = pd.concat([percentageTable, pd.crosstab(dfMDD['change_dsym_cut'], 
                                                                    dfMDD['change_'+label+'_cut'],
                                                                  normalize='all')], axis = 1)
    #Rearrange contingency table
    rearrangeCont = {}
    for i in range(len(labels)-1):
        cont = [contingencyTable.iloc[1,2*i+1],
               contingencyTable.iloc[1,2*i],
               contingencyTable.iloc[0, 2*i+1],
               contingencyTable.iloc[0,2*i]]
        rearrangeCont[labels[i+1]] = cont

    
    rearrangeCont = pd.DataFrame(rearrangeCont)
    locals()['rearrangeCont'+sample] = rearrangeCont

    #Create a mean table
    meanTable = {}
    for label in labels[1:]:
        mean = [dfMDD[(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 1)].imprv1.mean(),
                dfMDD[(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 0)].imprv1.mean(),
                dfMDD[(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 1)].imprv1.mean(),
                dfMDD[(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 0)].imprv1.mean()]
        meanTable[label] = mean
    meanTable = pd.DataFrame(meanTable)
    meanTable

    #Create a mean improvement table with sample size in parentheses
    meanNTable = meanTable.copy()
    for i in range(meanNTable.shape[0]):
        for j in range(meanNTable.shape[1]):
            meanNTable.iloc[i,j] = str(round(meanTable.iloc[i,j],1)) + ' (' + str(rearrangeCont.iloc[i,j]) +')'

    meanNTable = meanNTable.rename(index = {0:'both', 1:'dep', 2:'other', 3:'none'})
    print(sample)
    display(meanNTable)
    meanNTable.to_csv('../Results/'+sample+'_meanImprvNTable.csv') 

```

    MDDAll



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both</th>
      <td>3.2 (475)</td>
      <td>3.2 (438)</td>
      <td>3.3 (491)</td>
      <td>3.3 (416)</td>
      <td>3.3 (510)</td>
    </tr>
    <tr>
      <th>dep</th>
      <td>3.0 (159)</td>
      <td>3.0 (196)</td>
      <td>2.8 (143)</td>
      <td>2.9 (218)</td>
      <td>2.8 (124)</td>
    </tr>
    <tr>
      <th>other</th>
      <td>2.5 (102)</td>
      <td>2.7 (159)</td>
      <td>2.8 (142)</td>
      <td>2.8 (119)</td>
      <td>2.9 (176)</td>
    </tr>
    <tr>
      <th>none</th>
      <td>2.3 (390)</td>
      <td>2.1 (333)</td>
      <td>2.1 (350)</td>
      <td>2.2 (373)</td>
      <td>2.0 (316)</td>
    </tr>
  </tbody>
</table>
</div>


    MDDCurrPrin



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both</th>
      <td>3.2 (366)</td>
      <td>3.2 (335)</td>
      <td>3.3 (375)</td>
      <td>3.3 (317)</td>
      <td>3.3 (393)</td>
    </tr>
    <tr>
      <th>dep</th>
      <td>2.9 (117)</td>
      <td>3.0 (148)</td>
      <td>2.7 (108)</td>
      <td>2.9 (166)</td>
      <td>2.7 (90)</td>
    </tr>
    <tr>
      <th>other</th>
      <td>2.4 (71)</td>
      <td>2.7 (108)</td>
      <td>2.8 (99)</td>
      <td>2.8 (89)</td>
      <td>2.9 (126)</td>
    </tr>
    <tr>
      <th>none</th>
      <td>2.2 (290)</td>
      <td>2.1 (253)</td>
      <td>2.1 (262)</td>
      <td>2.1 (272)</td>
      <td>2.0 (235)</td>
    </tr>
  </tbody>
</table>
</div>


### Compute the Percentage of Higher Global Patient Ratings of Improvement in Each Responder Group


```python
for sample in ['MDDAll', 'MDDCurrPrin']:
    dfMDD = locals().get('rdqImprvDxComp'+sample)
    rearrangeCont = locals()['rearrangeCont'+sample]
    
    #Dichotimized Improvement scores (3,4 = higher improvement) 
    #Compute the number of people with higher improvement socres in each responder group
    
    imprvN34Table = {}
    percTable = {}
    for label in labels[1:]:
        N = [sum(dfMDD[(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 1)].imprv1>=3),
                sum(dfMDD[(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 0)].imprv1>=3),
                sum(dfMDD[(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 1)].imprv1>=3),
                sum(dfMDD[(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 0)].imprv1>=3)]

        imprvN34Table[label] = N

    imprvN34Table = pd.DataFrame(imprvN34Table)
    
    #Compute the percentage of people with higher improvement scores in each responder group
    #Number of people with improvement ratings of 3 or 4 in each group/Number of responders in each group
    
    imprvPercNTable = imprvN34Table.copy()

    for i in range(imprvPercNTable.shape[0]):
        for j in range(imprvPercNTable.shape[1]):
            imprvPercNTable.iloc[i,j] = str(round(imprvN34Table.iloc[i,j]/rearrangeCont.iloc[i,j]*100,1)) + '% (' + str(rearrangeCont.iloc[i,j]) + ')'
    
    imprvPercNTable  = imprvPercNTable.rename(index = {0:'both', 1:'dep', 2:'other', 3:'none'})
    imprvPercNTable .to_csv('../Results/'+sample+'_imprvPercNTable.csv') 
    
    print(sample)
    display(imprvPercNTable)    
```

    MDDAll



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both</th>
      <td>86.3% (475)</td>
      <td>86.3% (438)</td>
      <td>89.4% (491)</td>
      <td>87.3% (416)</td>
      <td>87.5% (510)</td>
    </tr>
    <tr>
      <th>dep</th>
      <td>76.1% (159)</td>
      <td>78.1% (196)</td>
      <td>64.3% (143)</td>
      <td>77.1% (218)</td>
      <td>68.5% (124)</td>
    </tr>
    <tr>
      <th>other</th>
      <td>54.9% (102)</td>
      <td>67.3% (159)</td>
      <td>70.4% (142)</td>
      <td>68.1% (119)</td>
      <td>74.4% (176)</td>
    </tr>
    <tr>
      <th>none</th>
      <td>45.4% (390)</td>
      <td>37.8% (333)</td>
      <td>38.0% (350)</td>
      <td>40.8% (373)</td>
      <td>32.3% (316)</td>
    </tr>
  </tbody>
</table>
</div>


    MDDCurrPrin



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both</th>
      <td>85.8% (366)</td>
      <td>86.0% (335)</td>
      <td>89.1% (375)</td>
      <td>87.1% (317)</td>
      <td>86.8% (393)</td>
    </tr>
    <tr>
      <th>dep</th>
      <td>76.1% (117)</td>
      <td>77.7% (148)</td>
      <td>63.9% (108)</td>
      <td>76.5% (166)</td>
      <td>68.9% (90)</td>
    </tr>
    <tr>
      <th>other</th>
      <td>50.7% (71)</td>
      <td>65.7% (108)</td>
      <td>69.7% (99)</td>
      <td>67.4% (89)</td>
      <td>76.2% (126)</td>
    </tr>
    <tr>
      <th>none</th>
      <td>44.8% (290)</td>
      <td>37.5% (253)</td>
      <td>37.0% (262)</td>
      <td>39.0% (272)</td>
      <td>29.8% (235)</td>
    </tr>
  </tbody>
</table>
</div>


### Pairwise Comparisons between Each Responder Group

We can either first do an ANOVA followed by post-hoc pairwise comparisons for those with significant ANOVA results or directly perform pairwise comparisons. Given the large sample size in the current case, I would predict all ANOVA results to be significant. I would usually skip ANOVA in this case. Here, I am only putting down ANOVA code for demonstration purposes. 


```python
#ANOVA
for sample in ['MDDAll', 'MDDCurrPrin']:
    dfMDD = locals().get('rdqImprvDxComp'+sample)

    #Create responder group labels
    for label in labels[1:]:
        dfMDD['respGroup_'+label] = 0
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 1)] = 'both'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 0)] = 'dep'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 1)] = 'other'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 0)] = 'none'

    anovaTable = pd.DataFrame()
    for label in labels[1:]:
        model = ols('imprv1 ~ C(respGroup_'+label+')', data=dfMDD).fit()
        anova = sm.stats.anova_lm(model, type = 2)
        anova = anova[['sum_sq','df','mean_sq','F','PR(>F)']] 
        anovaTable = pd.concat([anovaTable, anova])
    print(sample)
    display(anovaTable)
```

    MDDAll



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sum_sq</th>
      <th>df</th>
      <th>mean_sq</th>
      <th>F</th>
      <th>PR(&gt;F)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>C(respGroup_ndsym)</th>
      <td>213.646289</td>
      <td>3.0</td>
      <td>71.215430</td>
      <td>84.645732</td>
      <td>2.260267e-49</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>943.978045</td>
      <td>1122.0</td>
      <td>0.841335</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_cope)</th>
      <td>245.064480</td>
      <td>3.0</td>
      <td>81.688160</td>
      <td>100.436278</td>
      <td>1.369894e-57</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>912.559854</td>
      <td>1122.0</td>
      <td>0.813333</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_pmh)</th>
      <td>280.279624</td>
      <td>3.0</td>
      <td>93.426541</td>
      <td>119.479354</td>
      <td>3.780155e-67</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>877.344710</td>
      <td>1122.0</td>
      <td>0.781947</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_fun)</th>
      <td>251.285972</td>
      <td>3.0</td>
      <td>83.761991</td>
      <td>103.693011</td>
      <td>2.987838e-59</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>906.338362</td>
      <td>1122.0</td>
      <td>0.807788</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_well)</th>
      <td>302.513540</td>
      <td>3.0</td>
      <td>100.837847</td>
      <td>132.310415</td>
      <td>2.187827e-73</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>855.110793</td>
      <td>1122.0</td>
      <td>0.762131</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


    MDDCurrPrin



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>sum_sq</th>
      <th>df</th>
      <th>mean_sq</th>
      <th>F</th>
      <th>PR(&gt;F)</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>C(respGroup_ndsym)</th>
      <td>170.474255</td>
      <td>3.0</td>
      <td>56.824752</td>
      <td>67.256812</td>
      <td>5.535918e-39</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>709.709395</td>
      <td>840.0</td>
      <td>0.844892</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_cope)</th>
      <td>192.973675</td>
      <td>3.0</td>
      <td>64.324558</td>
      <td>78.626084</td>
      <td>7.826022e-45</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>687.209974</td>
      <td>840.0</td>
      <td>0.818107</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_pmh)</th>
      <td>223.584816</td>
      <td>3.0</td>
      <td>74.528272</td>
      <td>95.345507</td>
      <td>4.108362e-53</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>656.598833</td>
      <td>840.0</td>
      <td>0.781665</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_fun)</th>
      <td>208.028897</td>
      <td>3.0</td>
      <td>69.342966</td>
      <td>86.658751</td>
      <td>7.399623e-49</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>672.154752</td>
      <td>840.0</td>
      <td>0.800184</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
    <tr>
      <th>C(respGroup_well)</th>
      <td>248.148565</td>
      <td>3.0</td>
      <td>82.716188</td>
      <td>109.933135</td>
      <td>4.801290e-60</td>
    </tr>
    <tr>
      <th>Residual</th>
      <td>632.035084</td>
      <td>840.0</td>
      <td>0.752423</td>
      <td>NaN</td>
      <td>NaN</td>
    </tr>
  </tbody>
</table>
</div>


As expected, we see significant group differences in global patient improvement ratings in each outcome domain.

Next, we perform pairwise comparisons based on t-tests. I compute the p-values of t-tests and cohen's d for effect size, which is independent of sample size. 


```python
#Fucntion to compute Cohen's d for effect size
def cohend(d1, d2):
    n1, n2 = len(d1), len(d2)
    s1, s2 = np.var(d1, ddof=1), np.var(d2, ddof=1)
    s = np.sqrt(((n1 - 1) * s1 + (n2 - 1) * s2) / (n1 + n2 - 2))
    u1, u2 = np.mean(d1), np.mean(d2)
    return (u1 - u2) / s
```


```python
for sample in ['MDDAll', 'MDDCurrPrin']:
    dfMDD = locals().get('rdqImprvDxComp'+sample)

    #Create responder group labels
    for label in labels[1:]:
        dfMDD['respGroup_'+label] = 0
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 1)] = 'both'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 1) & (dfMDD['change_'+label+'_cut'] == 0)] = 'dep'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 1)] = 'other'
        dfMDD['respGroup_'+label][(dfMDD['change_dsym_cut'] == 0) & (dfMDD['change_'+label+'_cut'] == 0)] = 'none'

    dfMDD['imprv_cut'] = pd.cut(dfMDD['imprv1'], [-98,2,98], labels = [0,1]) #Dichotimized improvement scores
    dfMDD['imprv_cut'] = dfMDD['imprv_cut'].astype('float') 

    respGroup = ['both','dep','other','none'] 
    sigTable = {}
    sigTableCut= {}

    for label in labels[1:]:
        sig = ['N']*6
        k = 0
        sig_cut = ['N']*6
        rowname = []
        for i in range(len(respGroup)):
            for j in range(i+1, len(respGroup)):   
                
                #t-test for the mean of improvement scores between responder groups
                res = ttest_ind(dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[i], 'imprv1'],
                         dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[j], 'imprv1']) 
                #cohen's d for the mean of improvement scores between responder groups
                d = cohend(dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[i], 'imprv1'],
                         dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[j], 'imprv1']) 

                #t-test for the dichotimized improvement scores between responder groups
                res_c = ttest_ind(dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[i], 'imprv_cut'],
                         dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[j], 'imprv_cut'])
                #cohen's d for the dichotimized improvement scores between responder groups
                d_c = cohend(dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[i], 'imprv_cut'],
                         dfMDD.loc[dfMDD['respGroup_'+label] == respGroup[j], 'imprv_cut'])


                #Record p-values
                sig[k] = 'p=' +str(round(res[1],3)) + '; d='+str(round(d,2))
                sig_cut[k] = 'p=' +str(round(res_c[1],3)) + '; d='+str(round(d_c,2))

                k+=1

                rowname.append(respGroup[i]+ '-' +respGroup[j])

        sigTable[label] = sig
        sigTableCut[label] = sig_cut


    renameIndex = {}
    for i in range(len(rowname)):
            renameIndex[i] = rowname[i]

    sigTable = pd.DataFrame(sigTable)
    sigTable = sigTable.rename(index = renameIndex)
    sigTable.to_csv('../Results/'+sample+'meanImprvPairWiseComp.csv')

    sigTableCut = pd.DataFrame(sigTableCut).rename(index = renameIndex)
    sigTableCut.to_csv('../Results/'+sample+'percImprvPairWiseComp.csv')
    
    print(sample,': Continuous Improvement Scores')
    display(sigTable)
    print(sample,': Dichotimized Improvement Scores')
    display(sigTableCut)
```

    MDDAll : Continuous Improvement Scores



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both-dep</th>
      <td>p=0.0; d=0.37</td>
      <td>p=0.004; d=0.25</td>
      <td>p=0.0; d=0.71</td>
      <td>p=0.0; d=0.47</td>
      <td>p=0.0; d=0.64</td>
    </tr>
    <tr>
      <th>both-other</th>
      <td>p=0.0; d=0.89</td>
      <td>p=0.0; d=0.59</td>
      <td>p=0.0; d=0.6</td>
      <td>p=0.0; d=0.62</td>
      <td>p=0.0; d=0.52</td>
    </tr>
    <tr>
      <th>both-none</th>
      <td>p=0.0; d=1.05</td>
      <td>p=0.0; d=1.22</td>
      <td>p=0.0; d=1.33</td>
      <td>p=0.0; d=1.2</td>
      <td>p=0.0; d=1.42</td>
    </tr>
    <tr>
      <th>dep-other</th>
      <td>p=0.0; d=0.47</td>
      <td>p=0.002; d=0.33</td>
      <td>p=0.518; d=-0.08</td>
      <td>p=0.128; d=0.17</td>
      <td>p=0.394; d=-0.1</td>
    </tr>
    <tr>
      <th>dep-none</th>
      <td>p=0.0; d=0.67</td>
      <td>p=0.0; d=0.93</td>
      <td>p=0.0; d=0.63</td>
      <td>p=0.0; d=0.78</td>
      <td>p=0.0; d=0.75</td>
    </tr>
    <tr>
      <th>other-none</th>
      <td>p=0.026; d=0.25</td>
      <td>p=0.0; d=0.6</td>
      <td>p=0.0; d=0.68</td>
      <td>p=0.0; d=0.59</td>
      <td>p=0.0; d=0.84</td>
    </tr>
  </tbody>
</table>
</div>


    MDDAll : Dichotimized Improvement Scores



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both-dep</th>
      <td>p=0.002; d=0.28</td>
      <td>p=0.009; d=0.22</td>
      <td>p=0.0; d=0.71</td>
      <td>p=0.001; d=0.28</td>
      <td>p=0.0; d=0.52</td>
    </tr>
    <tr>
      <th>both-other</th>
      <td>p=0.0; d=0.84</td>
      <td>p=0.0; d=0.5</td>
      <td>p=0.0; d=0.55</td>
      <td>p=0.0; d=0.52</td>
      <td>p=0.0; d=0.36</td>
    </tr>
    <tr>
      <th>both-none</th>
      <td>p=0.0; d=0.97</td>
      <td>p=0.0; d=1.18</td>
      <td>p=0.0; d=1.31</td>
      <td>p=0.0; d=1.12</td>
      <td>p=0.0; d=1.42</td>
    </tr>
    <tr>
      <th>dep-other</th>
      <td>p=0.0; d=0.46</td>
      <td>p=0.023; d=0.24</td>
      <td>p=0.275; d=-0.13</td>
      <td>p=0.073; d=0.21</td>
      <td>p=0.265; d=-0.13</td>
    </tr>
    <tr>
      <th>dep-none</th>
      <td>p=0.0; d=0.64</td>
      <td>p=0.0; d=0.87</td>
      <td>p=0.0; d=0.54</td>
      <td>p=0.0; d=0.78</td>
      <td>p=0.0; d=0.78</td>
    </tr>
    <tr>
      <th>other-none</th>
      <td>p=0.087; d=0.19</td>
      <td>p=0.0; d=0.61</td>
      <td>p=0.0; d=0.68</td>
      <td>p=0.0; d=0.56</td>
      <td>p=0.0; d=0.92</td>
    </tr>
  </tbody>
</table>
</div>


    MDDCurrPrin : Continuous Improvement Scores



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both-dep</th>
      <td>p=0.0; d=0.4</td>
      <td>p=0.002; d=0.31</td>
      <td>p=0.0; d=0.78</td>
      <td>p=0.0; d=0.54</td>
      <td>p=0.0; d=0.67</td>
    </tr>
    <tr>
      <th>both-other</th>
      <td>p=0.0; d=1.0</td>
      <td>p=0.0; d=0.65</td>
      <td>p=0.0; d=0.64</td>
      <td>p=0.0; d=0.64</td>
      <td>p=0.0; d=0.48</td>
    </tr>
    <tr>
      <th>both-none</th>
      <td>p=0.0; d=1.08</td>
      <td>p=0.0; d=1.24</td>
      <td>p=0.0; d=1.36</td>
      <td>p=0.0; d=1.28</td>
      <td>p=0.0; d=1.48</td>
    </tr>
    <tr>
      <th>dep-other</th>
      <td>p=0.001; d=0.53</td>
      <td>p=0.01; d=0.33</td>
      <td>p=0.498; d=-0.09</td>
      <td>p=0.332; d=0.13</td>
      <td>p=0.245; d=-0.16</td>
    </tr>
    <tr>
      <th>dep-none</th>
      <td>p=0.0; d=0.66</td>
      <td>p=0.0; d=0.9</td>
      <td>p=0.0; d=0.61</td>
      <td>p=0.0; d=0.8</td>
      <td>p=0.0; d=0.79</td>
    </tr>
    <tr>
      <th>other-none</th>
      <td>p=0.18; d=0.18</td>
      <td>p=0.0; d=0.58</td>
      <td>p=0.0; d=0.67</td>
      <td>p=0.0; d=0.64</td>
      <td>p=0.0; d=0.93</td>
    </tr>
  </tbody>
</table>
</div>


    MDDCurrPrin : Dichotimized Improvement Scores



<div>
<style scoped>
    .dataframe tbody tr th:only-of-type {
        vertical-align: middle;
    }

    .dataframe tbody tr th {
        vertical-align: top;
    }

    .dataframe thead th {
        text-align: right;
    }
</style>
<table border="1" class="dataframe">
  <thead>
    <tr style="text-align: right;">
      <th></th>
      <th>ndsym</th>
      <th>cope</th>
      <th>pmh</th>
      <th>fun</th>
      <th>well</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <th>both-dep</th>
      <td>p=0.014; d=0.26</td>
      <td>p=0.024; d=0.22</td>
      <td>p=0.0; d=0.7</td>
      <td>p=0.003; d=0.29</td>
      <td>p=0.0; d=0.49</td>
    </tr>
    <tr>
      <th>both-other</th>
      <td>p=0.0; d=0.93</td>
      <td>p=0.0; d=0.53</td>
      <td>p=0.0; d=0.56</td>
      <td>p=0.0; d=0.53</td>
      <td>p=0.005; d=0.29</td>
    </tr>
    <tr>
      <th>both-none</th>
      <td>p=0.0; d=0.97</td>
      <td>p=0.0; d=1.17</td>
      <td>p=0.0; d=1.33</td>
      <td>p=0.0; d=1.16</td>
      <td>p=0.0; d=1.47</td>
    </tr>
    <tr>
      <th>dep-other</th>
      <td>p=0.0; d=0.55</td>
      <td>p=0.034; d=0.27</td>
      <td>p=0.378; d=-0.12</td>
      <td>p=0.119; d=0.21</td>
      <td>p=0.235; d=-0.16</td>
    </tr>
    <tr>
      <th>dep-none</th>
      <td>p=0.0; d=0.65</td>
      <td>p=0.0; d=0.87</td>
      <td>p=0.0; d=0.56</td>
      <td>p=0.0; d=0.81</td>
      <td>p=0.0; d=0.85</td>
    </tr>
    <tr>
      <th>other-none</th>
      <td>p=0.375; d=0.12</td>
      <td>p=0.0; d=0.58</td>
      <td>p=0.0; d=0.68</td>
      <td>p=0.0; d=0.59</td>
      <td>p=0.0; d=1.04</td>
    </tr>
  </tbody>
</table>
</div>


Overall, you can see that patients with 50% improvement in depressive symptom reduction and other outcome domains reported significantly higher global ratings of improvement than patients with improvements in only one outcome domain. Patients with 50% improvement in either depressive symptom reduction or other outcome domains reported higher global improvement ratings than nonresponders. Lastly, there was no significant difference in global improvement ratings between patients who only responded to depressive symptom reduction or other treatment outcome domains. 

<b> Such findings indicate that the improvement in symptom reduction is no more critical than other treatment outcome domains. We should consider adding other treatment outcome domains to standard outcome metrics to capture the full spectrum of depression remission. </b>


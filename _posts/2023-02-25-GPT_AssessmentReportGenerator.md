---
layout: post
title: "GPT-Powered Assessment Report Generator"
categories: GenerativeAI
excerpt: Writing psychodiagnostic reports can be a cumbersome and time-consuming process. It can take up to four hours to write a full report, which is an essential component of patient care management. Writing reports in clear, neutral medical language is important, especially since patients tend to be transferred between different levels of care. However, Natural Language Processing (NLP) has the potential to automate the report generation process, saving time and effort for healthcare providers. Here is a quick demo of how to combine NLP with medical report generation. 
---

## Introduction
Writing psychodiagnostic reports can be a cumbersome and time-consuming process. It can take up to four hours to write a full report, which is an essential component of patient care management. Writing reports in clear, neutral medical language is important, especially since patients tend to be transferred between different levels of care. However, Natural Language Processing (NLP) has the potential to automate the report generation process, saving time and effort for healthcare providers.

## Installation

To get started with NLP-powered report generation, you'll need to create a Python virtual environment (venv) and install Flask and OpenAI for Python. You'll also need to add the OpenAI API to the environment.

1. Create a Python virtual environment using the following command:
```
python3 -m venv myenv
```

2. Activate the virtual environment using the following command:
```
. myenv/bin/activate
```

3. Install Flask using the following command:
```
pip install Flask
```

4. Install the OpenAI Python package using the following command:
```
pip install openai
```

5. Add your OpenAI API key to the environment using the following command:
```
export OPENAI_API_KEY=<your-api-key>
```

6. Create a Python Flask file, e.g., app.py, in your preferred code editor.


## Usage:

Once you've completed the installation process, you can start using NLP to generate psychodiagnostic reports automatically. In your Flask app, you can use OpenAI's GPT-3 language model to generate reports based on input data. Here's an example of how you can generate a report using OpenAI:

```
import openai
openai.api_key = os.environ["OPENAI_API_KEY"]

def generate_report(input_data):
    prompt = "Generate a psychodiagnostic report for the following patient:\n" + input_data
    response = openai.Completion.create(
        engine="text-davinci-002",
        prompt=prompt,
        max_tokens=2048,
        n=1,
        stop=None,
        temperature=0.5,
    )
    report = response.choices[0].text
    return report
```

## Demo

Below is a demo of a report interface I created for assessing depression. 


![Fig1](/images/GenerativeAI/interface.png)

<br>
Once you click ```sumbit```, the report generator based on GPT will create a report written in professional medical tone following the diagnostic rule of MDD in the DSM-5. The whoe process tkaes less than 5 minutes, while typing the report on your own can easily take 15-30 minutes, depending on your familiarity with the medical lanaguage. 

![Fig2](/images/GenerativeAI/report.png)

You can find my code in my [gituhub rep](https://github.com/sin-ying-lin/assessment_report_generator/).

To use the package. Please follow the steps below. 

1. Clone my GitHub repository. 
```
git clone https://github.com/sin-ying-lin/assessment_report_generator/
```

2. Once you move into the cloned folder. Use the following command to activate the environemnt. 
```
. generator/bin/activate
```

3. Add your OpenAI API key to the environment using the following command:
```
export OPENAI_API_KEY=<your-api-key>
```

4. Lauch report generator
```
python generator.py
```
5. Copy the locohost address to your browser

6. Try insert different informatino to the report generator interface. Click submit to generate a reprot. 

Assessment for other common metnal disorders will be rolled out soon.

## Conclusion

By using NLP and OpenAI's powerful language model, you can automate the process of generating psychodiagnostic reports, saving time and effort for healthcare providers. With the help of Flask, you can create a user-friendly interface that allows users to input patient data and receive a generated report in return. With this technology, you can improve patient care management and ensure that patients receive the care they need, no matter where they are transferred.

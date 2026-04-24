<img width="1658" height="954" alt="image" src="https://github.com/user-attachments/assets/6110c435-81cf-4b66-b9a5-ab4a3563b02e" />
<img width="1434" height="613" alt="image" src="https://github.com/user-attachments/assets/b671957a-2ec6-4c1c-9320-58b5cc4d5c84" />


import streamlit as st
from langchain_openai import ChatOpenAI
from langchain_core.prompts import PromptTemplate
from langchain_core.output_parsers import StrOutputParser
import os
from dotenv import load_dotenv

load_dotenv()

llm = ChatOpenAI(model_name="gpt-3.5-turbo", temperature=0.2, api_key=os.getenv("OPENAI_API_KEY"))

prompt = PromptTemplate(
    input_variables=["Code_task"],
    template="""
    You are a proffessional pro level coding assistant. Help the user with the following task
    {Code_task} 
    provide a clean, neat and well organized code and explanations wherever applicable
    Your Response:
    """
)

chain = prompt | llm | StrOutputParser()

st.title("Code Assistant")

code_task = st.text_input("Enter the code task")

if st.button("Submit"):
    if code_task.strip() == "":
        st.warning("Please allocate me a task")
    else:
        response = chain.invoke({"Code_task": code_task})
        st.subheader("Response :")
        st.code(response, language="python")

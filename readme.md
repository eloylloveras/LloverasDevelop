> Created by @ONCOGENIA group // Hospital General de Granollers - 16/11/2024


> [!IMPORTANT]
> All documents related to this documentation are uploaded here: [[insert-link-results-folder-from-amazon](https://us-west-2.console.aws.amazon.com/s3/buckets/results-clinic-hackathon-2024?prefix=Team9/Optimizing%20Precision%20Oncology%20in%20the%20Clinic%20Comprehensive%20Cancer%20Center%20with%20Generative%20AI%20(OPERA%20Project)/&region=us-west-2&bucketType=general)]. <br>
> Our application has been programmed and validated following the legal requeriments documented on EUAIACTComplianceAndPolicyRecomendations. <br>

## 


<div align="center"><img src="https://github.com/user-attachments/assets/725f8758-59f2-4a0e-8a61-9fcaec256576" width="900" height="400"/></div>

<br>


# SUMMARY

This document explains how to set up a the _**Optimizing Precision Oncology in the Clinic Comprehensive Cancer Center with Generative AI**_ challenge proposed in the Event using Amazon Web Server (AWS) Services with a RAG structure.

The project is divided in 3 phases given a patientID:
1. **PHASE1:** Consist in getting a summary from the clinic notes. (We will get the summary in XML format for his easiest treatment)
2. **PHASE2:** With the data of the summary we obtain the cancer subtype and the stage and we propose a treatment based on European Guidlines EOMO plus recomendations generated based on the Special Situations Guidlines.
3. **PHASE3:** With clinical trials obtained as a JSON from the Website Clinical Trials.gov (https://clinicaltrials.gov/) and the summary obtainend in phase1 we will get a list of possible trials based on the patient. 


<br>

The Phases are structured by the following: 
* Prompt Template 
* Generic Method to comunicate to the model provided with his Phase Prompt Template and his correspondant Bedrock Knowledge Base ID
* Answer 

<br>

We will use the following services from Amazon Web Services:

* **_Amazon S3 bucket_** (Knowledge Files Directory)
* **_Bedrock Knowledge Base_** (Knowledge Base from an S3 Bucket)
* **_AI Models from Bedrock_** ([Anthropic Claude 3.5 Sonnet V2](https://us-west-2.console.aws.amazon.com/bedrock/home?region=us-west-2#/providers?model=anthropic.claude-3-5-sonnet-20241022-v2:0) and [Cohere Embed Multilingual v3](https://us-west-2.console.aws.amazon.com/bedrock/home?region=us-west-2#/providers?model=cohere.embed-multilingual-v3))
* **_CloudWatch_** (Logs to have a register of all Data asked,generated... to the AI models )
* **_SageMaker_** (JupyterLab, programming-environment. (We willset up a Python 3.12 kernel, the default version is 3.10))

<br>

We will use the following libraries using **_Python_** as the program lenguage to integrate all these services together.
1. **Boto3**
1. **Pandas**
1. **io**
1. **xml.etree.ElementTree**
1. **re**
 
<br>


# PHASE SUMMARIES

> [!IMPORTANT]
> All the Files, Buckets, Bedrock Bases, Prompt Templates mentioned in the following phases summaries will be documented on the end of this document.

<br>

## PHASE 1
### Summary 
1. With the Patient ID provided ("In the code is a variable but the main ideal is to have a Python Flask Aplication with a UI Interface") we will search his medical report on the **[DAT30_01](https://us-west-2.console.aws.amazon.com/bedrock/home?region=us-west-2#/knowledge-bases:~:text=%2D-,DAT30_01,-Disponible)** with Knowledge Base id '9LU9RAZUQL' provided with the knowledge from bucket **BASE30/[DAT30_01/](https://us-west-2.console.aws.amazon.com/s3/buckets/data-clinic-hackathon-2024?region=us-west-2&bucketType=general&prefix=BASE30/DAT30_01/&showversions=false)**.

1. With the medical report from the patient and the promp template from Phase1 (Phase1-Prompt-Template documented at the end of the document) we will generate a Summary in XML format (In xml format to get the variables more easy and at the same time to provide a more easy comprehension to the AI Model) using the generic method **generate_response** and the KNOWLEDGE_BASE_ID 'EXFAKCJDH9' a Knowledge Base provided with Phase1 Examples. 

1. With the Generated-Summary in XML format we will make the same summary but in Natural Lenguage. (This one is gona be provided to the doctor)


```python

# COLLECT FILENAMES FROM FOLDER
BUCKET01_PATH = "BASE30/DAT30_01/"

s3_client = boto3.client('s3')
paginator = s3_client.get_paginator('list_objects')
response_iterator = paginator.paginate(Bucket=BUCKET_NAME, Prefix=BUCKET01_PATH)
for res in response_iterator:
    conts = res['Contents']
    keys =  [cont['Key'] for cont in conts]
    break
    
file_names = [key.split("/")[-1] for key in keys]

# EXTRACT ID FROM FILENAME
def extract_patient_id(filename):
    # Expresión regular para capturar la cuarta secuencia numérica en el nombre del archivo
    match = re.search(r'^\d+_\d+_\w+_\w+_(\d+)_\d+_\d+', filename)
    if match:
        return match.group(1)
    return None

# CREATE DICTIONARY OF USERID MAPPED TO FILENAME
file_dict = {}
for fname in file_names:
    curr_id = extract_patient_id(fname)
    file_dict[curr_id] = fname

def read_txt_from_bucket(bucket_name, file_name):
    try:
        s3_client = boto3.client('s3')
        
        # Obtain S3 object
        response = s3_client.get_object(Bucket=bucket_name, Key=file_name)

        # Read the file content as bytes
        content = response['Body'].read()

        return content

    except Exception as e:
        print(f"Error al leer el archivo desde el bucket: {e}")
        return None

#  CODE TO GET USER ID INFORMATION FROM 01 

user_note = read_txt_from_bucket(BUCKET_NAME, BUCKET01_PATH + file_dict[USER_ID])

phase1_prompt_template = """ documented at the end of the document """

def generate_response(query: str, model: str, prompt: str) -> str:

    response = bedrock_agent_runtime_client.retrieve_and_generate(
        input={
            'text': query,
        },
        retrieveAndGenerateConfiguration={
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': KNOWLEDGE_BASE_ID,
                'modelArn': model,
                'generationConfiguration': {
                    'promptTemplate': {
                        'textPromptTemplate': prompt
                    }
                }
            },
            'type': 'KNOWLEDGE_BASE'
        }
    )

    return response["output"]["text"]

# GET THE NATURAL LENGUAGE SUMMARY
summary_response_natural_lenguage = generate_response(query = "Give me the result in Natural Lenguage.", model = GENERATION_MODEL, prompt = phase1_prompt_template)
DATA_NL = summary_response_natural_lenguage

# GET THE SUMMARY IN XML FORMAT
response = generate_response(query = "Answer must start with the <data> TAG , nothing else", model = GENERATION_MODEL, prompt = prompt )
DATA = response

```

<br>

## PHASE 2
### Summary
1. With the Summary generated in phase1 in XML , we will save the following data: 
* Subtipo
* EstadioActual
* Metastasis

2. We gona execute a decision tree based on the data obtained before to load one or another data to the generic-method knowledge base. 
> If EstadioActual between the numbers 1 to 3 then we will load the knowledge base **EstadioGuidlines** with ID 'WKLGRLDZZS' with the knowledge provided by the **extrainfobucket**. 

> If EstadioActual EQ 4 or Metastasis NOT EQ 'NULL' then we will check if Subtipo is 'SCLC' or 'NSCLCL'. If 'NSCLCL' then we will load the knowledge base **NSCLCL** with ID 'YCGTJ9OF2Q' provided with **guidlinesbucket/combinepdf** and **extendedguidlinebucket** knowledge, else we will load the **SCLC** with ID 'UK3QFV2VTK' provided with **guidlinesbucket/SCLC** and **extendedguidlinebucket** knowledge. 

> If EstadioActual and Metastasis are NULL then we will finish the program with a message like: 'No se ha encontrado Estadio entonces imposible generar tratamineto' // 'No stage found so it is impossible to generate treatment'

3. Using the generic method **generate_response** we will comunicate with the model using the Phase 2 Prompt Template loaded with the data obtained befores and with the correspondant KNOWLEDGE_BASE_ID. 

```python
root = ET.fromstring(DATA)

# Encontrar y obtener el valor de la etiqueta Subtipo
subtipo = root.find('.//Subtipo').text
print("Subtipo:", subtipo)

estadio = root.find('.//EstadioActual').text
print("Estadio:", estadio)

metastasis = root.find('.//mestastasi').text
print("Metastasis:", metastasis)

fin_programa = False
KNOWLEDGE_ID_GUIDLINES = "YCGTJ9OF2Q"
if estadio == 'I' or estadio == 'II' or estadio == 'III':
    KNOWLEDGE_ID_GUIDLINES = "WKLGRLDZZS"
elif estadio == 'IV' or metastasis != 'NULL':
    if 'SCLC' in subtipo:
        KNOWLEDGE_ID_GUIDLINES = "UK3QFV2VTK"
    else:
        KNOWLEDGE_ID_GUIDLINES = "YCGTJ9OF2Q"
else:
    #KNOWLEDGE_ID_GUIDLINES = 'EXFAKCJDH9'
    fin_programa = True
    

fase2_prompt_template = """ documented at the end of the document """

if fin_programa:
    print("No se ha encontrado Estadio entonces imposible generar tratamineto")
else:
    def generate_response2(query: str, model: str) -> str:

        response = bedrock_agent_runtime_client.retrieve_and_generate(
            input={
                'text': query,
            },
            retrieveAndGenerateConfiguration={
                'knowledgeBaseConfiguration': {
                    'knowledgeBaseId': f'{str(KNOWLEDGE_ID_GUIDLINES)}',
                    'modelArn': model,
                    'generationConfiguration': {
                        'promptTemplate': {
                            'textPromptTemplate': fase2_prompt_template
                        }
                    }
                },
                'type': 'KNOWLEDGE_BASE'
            }
        )

        return response["output"]["text"]
    
    response = generate_response2(query = "hola", model = GENERATION_MODEL)
    DATA_2 = response

```

<br>

## PHASE 3
### Summary
1. Obtain the clinical trials as a JSON from the Website Clinical Trials.gov (https://clinicaltrials.gov/) and saved in the **clintrials** bucket.
1. We will just comunicate to the model using the generic method 'generate_response' with the Phase 3 promp template and the knowledge base **clinical_trials** with ID '3ZL5WKBYIC' provided with **clintrials/trials_inclusion_exclusionTA (1).json** knowledge. 

```python

fase3_prompt_template = """ Documented at the end of the document """


CLINICAL_TRIALS_KNOWLEDGE = '3ZL5WKBYIC'

def generate_response2(query: str, model: str) -> str:

    response = bedrock_agent_runtime_client.retrieve_and_generate(
        input={
            'text': query,
        },
        retrieveAndGenerateConfiguration={
            'knowledgeBaseConfiguration': {
                'knowledgeBaseId': f'{str(CLINICAL_TRIALS_KNOWLEDGE)}',
                'modelArn': model,
                'generationConfiguration': {
                    'promptTemplate': {
                        'textPromptTemplate': fase3_prompt_template
                    }
                }
            },
            'type': 'KNOWLEDGE_BASE'
        }
    )

    return response["output"]["text"]

response = generate_response2(query = "hola", model = GENERATION_MODEL)
DATA_3 = response
```

## CODE TO GET ALL PHASES OUTPUTS
```python 
print("SUMMARY FOR THE DOCTOR\n")
print(DATA_NL)
print("\n==============================\n")
print("TEXT FROM FASE 2\n")
print(DATA_2)
print("\n==============================\n")
print("TEXT FROM FASE 3\n")
print(DATA_3)
```

<br>

## DOCUMENT'S LEGEND
 
1.	DAT30_01 (Knowledge Base ID’s: 9LU9RAZUQL)<br>
&emsp;a.	All the files here are the patiente clinic notes
2.	CLINTRIALBUCKET (Knowledge Base ID’s: 3ZL5WKBYIC )<br>
&emsp;a.	Trials_inclusion_exclusionTA.json (The trials obtained from the ClinicalTrials.gov )
3.	EXTENDEDGUIDLINESBUCKET (Knowledge Base ID’s: YCGTJ9OF2Q, UK3QFV2VTK)<br>
&emsp;a.	Formed by 11 pdf with Guidelines 
4.	EXTRAINFOBUCKET (Knowledge Base ID’s: WKLGRLDZZS )<br>
&emsp;a.	TractamentsEstadi1a3.pdf (Document where its defined different treatments for Estadi 1 to 3)<br>
5.	GUIDLINESBUCKET (Knowledge Base ID’s: YCGTJ9OF2Q, UK3QFV2VTK)<br>
&emsp;a.	Combinepdf.pdf (NSCLC_1.pdf and NSCLC_2.pdf combined)<br>
&emsp;b.	NSCLC_1.pdf  and NSCLC_2.pdf (Treatments for NSCLC Subtipo)<br>
&emsp;c.	SCLC.pdf (Treatments for SCLC Subtipo)<br>

## KNOWLEGE BASE LEGEND

1.	NSCLCL with ID YCGTJ9OF2Q
2.	SCLC with ID UK3QFV2VTK
3.	EstadioGuidlines with ID WKLGRLDZZS
4.	DAT30_01 with ID 9LU9RAZUQL 
5.	Clinical_trials with ID 3ZL5WKBYIC


# PROMPT TEMPLATES
## Prompt PHASE1
***
```python
prompt = f"""
Como Oncologa Experta a partir de un curso clínico (<CURSOCLINICO>) extrae la siguiente información y realiza un resumen de los datos clínicos en formato xml escribiendo para cada párrafo la información de cada sección. resumen,antecedentes_personales, intervenciones_quirurgicas,habitos_toxicos, ocupacion,social, antecedentes_cancer, historia_oncologica, pruebas_diagnosticas, estado_actual, exploracion_fisica.Debes mantener el nombre de los campos exactamente como se solicita en el prompy.
 

Te detallo lo que queremos que indiques para los ParametrizacionGuideline:("PS": "Informar de los PS que se escriben en las notas clínicas","Estadio": "Informar de los estadios que se escriben en las notas clínicas, aunque no especifique la paralabra estadio;"si no encuentras un valor solicitado, escribe NULL").
<data>
<resumen>
<Sexo></Sexo>
<Edad></Edad>
<Fecha_Diagnostico></Fecha_Diagnostico>
<Localizacion></Localizacion>
<Subtipo>Informar si es SCLC/CPCP, NSCLC/CPCNP o NULL</Subtipo>
<Histologia></Histologia>
<PSActual>Informar del PS o ECOG</PSActual>
<Tabaquismo></Tabaquismo>
<LocalizacionTumor></LocalizacionTumor>
<TNM></TNM>
<EstadioInicial></EstadioInicial>
<EstadioActual></EstadioActual>
<mestastasi></mestastasi>
<PerfilMolecular></PerfilMolecular>
<PDL1></PDL1>
</resumen>
 
<antecedentes_personales>
<alergias></alergias>
<enfermedades>
<endocrinas></endocrinas>
<digestivas></digestivas>
<neurologicas></neurologicas>
<reumatologicas></reumatologicas>
<infecciosas></infecciosas>
<oncologicas></oncologicas>
<hematologicas></hematologicas>
<hepaticas></hepaticas>
<renales></renales>
<dermatologicas></dermatologicas>
<oftalmologicas></oftalmologicas>
<otorrinolaringologicas></otorrinolaringologicas>
<ginecologicas></ginecologicas>
<urologicas></urologicas>
<psiquiatricas></psiquiatricas>
<traumatologicas></traumatologicas>
<respiratorias></respiratorias>
<cardiovasculares></cardiovasculares>
<autoinmunes></autoinmunes>
<trasplante></trasplante>
</enfermedades>
<intervenciones_quirurgicas></intervenciones_quirurgicas>
<medicacion_habitual></medicacion_habitual>
</antecedentes_personales>
 
<habitos_toxicos>
<tabaco>
<fumador>activo/exfumador/no_fumador</fumador>
<paquetes_ano></paquetes_ano>
<tipo>
<item>rubio</item>
<item>negro</item>
<item>puritos</item>
<item>puros</item>
</tipo>
<otros>
<cannabis>true</cannabis>
<cig_electronico>false</cig_electronico>
</otros>
<exposicion_pasiva>false</exposicion_pasiva>
</tabaco>
<alcohol>true</alcohol>
</habitos_toxicos>
 
<ocupacion></ocupacion>
 
<social>
<iabvd_independiente_actividades_basicas_vida_diaria></iabvd_independiente_actividades_basicas_vida_diaria>
<convivencia></convivencia>
</social>
 
<antecedentes_cancer>
<personales></personales>
<familiares>
<parentesco></parentesco>
<diagnostico></diagnostico>
</familiares>
</antecedentes_cancer>
 
<historia_oncologica>
<inicio_clinico_enfermedad>
<sintoma>informa si es sintoma, hallazgo o diagnostico</sintoma>
<fecha></fecha>
</inicio_clinico_enfermedad>
<pruebas_diagnosticas>
<analitica_sangre>
<marcadores_tumorales></marcadores_tumorales>
<fecha></fecha>
</analitica_sangre>
<rx_torax>
<resultado></resultado>
<fecha></fecha>
</rx_torax>
<tc_tap>
<resultado></resultado>
<fecha></fecha>
</tc_tap>
<tc_rm_cerebral>
<resultado></resultado>
<fecha></fecha>
</tc_rm_cerebral>
<pet_tc>
<resultado></resultado>
<fecha></fecha>
</pet_tc>
<funcion_respiratoria>
<fev1></fev1>
<cvf></cvf>
<fev1_cvf></fev1_cvf>
<dlco></dlco>
<prueba_broncodilatadora>false</prueba_broncodilatadora>
<fecha></fecha>
</funcion_respiratoria>
<broncoscopia>
<resultado></resultado>
<fecha></fecha>
</broncoscopia>
<ebus>
<resultado></resultado>
<fecha></fecha>
</ebus>
<mediastinoscopia>
<resultado></resultado>
<fecha></fecha>
</mediastinoscopia>
<biopsia>
<resultado></resultado>
<fecha></fecha>
</biopsia>
<anatomia_patologica>
<fecha></fecha>
<histologia></histologia>
<perfil_ihc></perfil_ihc>
</anatomia_patologica>
<estudio_molecular>
<fecha></fecha>
<ngs></ngs>
<resultado></resultado>
</estudio_molecular>
<pd-l1>
<porcentaje></porcentaje>
<fecha></fecha>
</pd-l1>
</pruebas_diagnosticas>
</historia_oncologica>
 
<estado_actual>
<sintomas>
<disnea></disnea>
<tos></tos>
<hemoptisis></hemoptisis>
<dolor_toracico></dolor_toracico>
<sintomas_generales>
<astenia></astenia>
<anorexia></anorexia>
<fiebre></fiebre>
<sudoracion></sudoracion>
</sintomas_generales>
<ecog_performance_status></ecog_performance_status>
</sintomas>
<exploracion_fisica>
<frecuencia_cardiaca></frecuencia_cardiaca>
<saturacion_o2></saturacion_o2>
<auscultacion></auscultacion>
</exploracion_fisica>
</estado_actual>
</data>

  $search_results$
  
<CURSOCLINICO>
 {user_note}
</CURSOCLINICO>
"""
```

## Prompt PHASE2
***
<br>

```python
fase2_prompt_template = f""" 
Eres una oncóloga con 20 años de experiencia. Utilizando las guidelines adjuntas como referencia y considerando los parámetros específicos del curso clínico proporcionado, elabora una propuesta de tratamiento detallada y razonada. Y añade recomendaciones basadas en las recomendaciones basadas en los sintomas que tiene el paciente o puede tener a partir del tratamiento propuesto. Desglosa cada decisión paso a paso, explicando la lógica detrás de cada recomendación y indica el documento y página del que extraes la información.

{DATA}
 $search_results$
"""
```
<br>

## Prompt PHASE3
***

```python
fase2_prompt_template = f""" 
Eres una oncóloga con 20 años de experiencia. Utilizando las guidelines adjuntas como referencia y considerando los parámetros específicos del curso clínico proporcionado, elabora una propuesta de tratamiento detallada y razonada. Y añade recomendaciones basadas en las recomendaciones basadas en los sintomas que tiene el paciente o puede tener a partir del tratamiento propuesto. Desglosa cada decisión paso a paso, explicando la lógica detrás de cada recomendación y indica el documento y página del que extraes la información.

{DATA}
 $search_results$
"""
```


<br>




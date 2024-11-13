# Azure Open AI o1-preview model - Supply Chain sourcing suppliers

## Introduction

- Goal is to show case how we can source the right suppliers for a given product.
- Based on quantity available, price, quality, and location.
- Wanted to bring the closest suppliers to the manufacturer.
- can avoid major disasters like tsunami, earthquake, etc.
- Can also avoid political issues, trade wars, etc.
- can also avoid issues like covid, and other country specific issues.
- Provide current constraints and future constraints.

## Pre-requisites

- Azure subscription
- Azure Open AI o1-preview model
- Local computer
- Visual Studio Code
- python 3.12 with virtual environment
- Make sure to store the endpoing and key in the .env file
- Model version to use: 2024-08-01-preview

## Steps

### Environment Setup

- create a .env file

```
AZURE_OPENAI_O1_ENDPOINT="https://api.openai.com"
AZURE_OPENAI_O1_KEY="yourkey"
```

### Code

- Create a new python file
- install necessary libraries
- Create .env file and setup the necessary keys
- import the libraries

```
import pandas as pd
import os

from pprint import pprint
from azure.ai.evaluation import evaluate
from openai import AzureOpenAI

from dotenv import load_dotenv
```

- Load the environment variables

```
load_dotenv()
```

- Initialize the Azure Open AI client

```
client = AzureOpenAI(
  azure_endpoint = os.getenv("AZURE_OPENAI_O1_ENDPOINT"), 
  api_key=os.getenv("AZURE_OPENAI_O1_KEY"),  
  api_version="2024-08-01-preview",
)
```

- Here is the function to process the supply chain sourcing
- i am including the data in the prompt as hardcoded sample dataset
- for real world application, add your own data and adjust the prompt

```
def processo1text(query):
    returntxt = ""

    rfttext = ""

    message_text = [
    {"role":"user", "content":f"""Here is the data for raw materials across the world, we are looking for silicon for ic chips for about 150000 in united states in texas plant, i need you to find the best vendors who can delivery in 2 weeks who are best to delivery on time. We want it in 10 days so that we can be ready for our manufacturing in 2 weeks. Consider the quantity in hand and see if they can fulfil our request, if one vendor can't then look for few vendor who can fulfil our request

   Here is the data set:OrderID,ProductID,RawMaterial,RawMaterialSupplier,SupplierCountry,Logistics_FromSupplier,ManufacturingPlant,PlantCountry,Logistics_ToWarehouse,Warehouse,WarehouseCountry,Logistics_ToStore,Store,StoreCountry,QuantityOnHand,DeliveryDate 1001,IC100,Silicon,Wenshan Minerals,China,DHL Express,TechFab Inc,Germany,FedEx Logistics,Central Depot NL,Netherlands,UPS Freight,ElectroMart,USA,523456,2023-11-12 1002,IC101,Copper,Andes Mining,Chile,Maersk Line,MicroWorks Ltd,Japan,Nippon Express,East Warehouse JP,Japan,Sagawa Express,TechStore,Australia,234567,2023-11-15 1003,IC102,Gold,GoldCo Mines,South Africa,Emirates Logistics,ChipMakers Inc,South Korea,Korea Post,Seoul Storage KR,South Korea,Lotte Global,DigitalHub,UK,345678,2023-11-18 1004,IC103,Silicon,Pacific Quartz,USA,UPS Freight,NanoTech Corp,Singapore,DHL Express,Harbor Warehouse SG,Singapore,SingPost,Electronica,Mexico,456789,2023-11-20 1005,IC104,Tantalum,Great Lakes Metals,Canada,Purolator,Elemental Tech,Germany,DHL Express,Rhine Warehouse DE,Germany,DB Schenker,Gadgets Galore,France,567890,2023-11-22 1006,IC105,Silver,Andes Silver Ltd,Peru,Latam Cargo,Precision Chips,China,SF Express,Shenzhen Storage CN,China,ZTO Express,Electronix,India,678901,2023-11-25 1007,IC106,Palladium,Norilsk Metals,Russia,TransContainer,NextGen Semis,USA,UPS Freight,Midwest Depot US,USA,USPS,TechBase,Germany,789012,2023-11-28 1008,IC107,Copper,Zambia Copper,Zambia,Kuehne + Nagel,Synapse Tech,UK,Royal Mail,London Storage UK,UK,DHL Parcel,DigiStore,Japan,890123,2023-12-01 1009,IC108,Silicon,Australian Silica,Australia,Toll Group,Integrated Circuits,Brazil,Correios,Brazil Central WH,Brazil,JadLog,ElectroHub,UAE,901234,2023-12-03 1010,IC109,Tantalum,DRC Metals,Congo,DHL Express,MicroChip Ltd,India,Blue Dart,Delhi Warehouse IN,India,Delhivery,TechWorld,China,12345,2023-12-05 1011,IC110,Gold,Australian Gold Inc,Australia,Qantas Freight,NanoElements,South Korea,Korea Post,Busan Storage KR,South Korea,Hanaro TNS,GadgetMart,USA,23456,2023-12-08 1012,IC111,Silver,Mexico Silver Co,Mexico,FedEx Logistics,Quantum Chips,Germany,DB Schenker,Berlin Warehouse DE,Germany,DHL Freight,Electronics4U,UK,34567,2023-12-10 1013,IC112,Palladium,Brazil Palladium,Brazil,Latam Cargo,HighTech Semis,Japan,Nippon Express,Osaka Storage JP,Japan,Yamato Logistics,TechSquare,Australia,45678,2023-12-12 1014,IC113,Silicon,Namibia Quartz,Namibia,CMA CGM,ChipTech Inc,Singapore,SingPost,West Warehouse SG,Singapore,Ninja Van,ElectroStore,Mexico,56789,2023-12-15 1015,IC114,Copper,Canadian Copper Inc,Canada,Air Canada Cargo,NanoChips Ltd,China,SF Express,Shanghai Storage CN,China,ZTO Express,GizmoShop,India,67890,2023-12-18 1016,IC115,Gold,Russian Gold Co,Russia,Aeroflot Cargo,Elemental Circuits,UK,British Airways Cargo,Manchester WH UK,UK,TNT Express,DigiMart,Germany,78901,2023-12-20 1017,IC116,Tantalum,Mozambique Metals,Mozambique,Kuehne + Nagel,MicroFab Inc,USA,UPS Freight,East Coast Depot US,USA,FedEx Ground,TechPoint,Japan,89012,2023-12-23 1018,IC117,Silver,Peruvian Silver,Peru,DHL Express,Semiconductor Co,Germany,DB Schenker,Hamburg Warehouse DE,Germany,DHL Freight,GadgetWorld,UAE,90123,2023-12-26 1019,IC118,Palladium,USA Palladium LLC,USA,UPS Freight,NanoCircuits Ltd,South Korea,Korea Post,Incheon Storage KR,South Korea,Hyundai Glovis,ElectroMart,Mexico,123456,2023-12-28 1020,IC119,Silicon,Brazil Silica,Brazil,Latam Cargo,MicroTech Inc,Japan,Nippon Express,Nagoya Warehouse JP,Japan,Sagawa Express,TechStore,China,234567,2023-12-30 1021,IC120,Copper,Chile Copper Corp,Chile,Compania Sud Americana,Chip Innovators,China,SF Express,Guangzhou Storage CN,China,YTO Express,DigitalHub,India,345678,2024-01-02 1022,IC121,Gold,Canadian Gold Ltd,Canada,Air Canada Cargo,Elemental Chips,Singapore,SingPost,Central Warehouse SG,Singapore,Ninja Van,Electronica,USA,456789,2024-01-04 1023,IC122,Tantalum,Rwanda Metals,Rwanda,Qatar Airways Cargo,NanoElements,Germany,DHL Express,Frankfurt Warehouse DE,Germany,DB Schenker,Gadgets Galore,UK,567890,2024-01-06 1024,IC123,Silver,Indonesian Silver,Indonesia,Garuda Cargo,Precision Semis,UK,Royal Mail,Birmingham WH UK,UK,DHL Parcel,DigitalHub,Australia,678901,2024-01-09 1025,IC124,Palladium,Australian Palladium,Australia,Qantas Freight,Advanced Circuits,Brazil,Correios,Sao Paulo WH BR,Brazil,JadLog,ElectroMart,UAE,789012,2024-01-12 1026,IC125,Silicon,Egyptian Silica,Egypt,EgyptAir Cargo,MicroNano Tech,India,Blue Dart,Mumbai Warehouse IN,India,Gati Ltd,TechWorld,China,890123,2024-01-15 1027,IC126,Copper,Mongolian Copper,Mongolia,Aeromongolia,NanoCircuits Ltd,South Korea,Korea Post,Daejeon Storage KR,South Korea,Lotte Global,DigiStore,USA,901234,2024-01-17 1028,IC127,Gold,Indonesian Gold PT,Indonesia,Garuda Cargo,ChipMakers Inc,Germany,DB Schenker,Bremen Warehouse DE,Germany,DHL Freight,Electronix,UK,12345,2024-01-20 1029,IC128,Tantalum,Nigerian Metals,Nigeria,DHL Express,SemiTech Co,Japan,Nippon Express,Kobe Warehouse JP,Japan,Yamato Logistics,GadgetMart,Mexico,23456,2024-01-22 1030,IC129,Silver,Kazakhstan Silver,Kazakhstan,Air Astana Cargo,Elemental Semis,Singapore,SingPost,North Warehouse SG,Singapore,Ninja Van,TechStore,India,34567,2024-01-25 1031,IC130,Palladium,South African Metals,South Africa,Emirates Logistics,NanoTech Corp,China,SF Express,Beijing Storage CN,China,ZTO Express,ElectroHub,Germany,45678,2024-01-28 1032,IC131,Silicon,USA Quartz Ltd,USA,UPS Freight,Precision Chips,UK,Royal Mail,Leeds Warehouse UK,UK,DHL Parcel,DigitalHub,Japan,56789,2024-01-31 1033,IC132,Copper,Polish Copper SA,Poland,LOT Cargo,Chip Innovations,Canada,Purolator,Toronto WH CA,Canada,Canada Post,GadgetWorld,UAE,67890,2024-02-02 1034,IC133,Gold,China Gold Co,China,China Post,Advanced Semis,USA,FedEx Logistics,West Coast Depot US,USA,UPS Ground,ElectroStore,Australia,78901,2024-02-05 1035,IC134,Tantalum,Argentinian Metals,Argentina,Aerolineas Cargo,NanoElements,Germany,DB Schenker,Stuttgart WH DE,Germany,DHL Freight,Electronica,Mexico,89012,2024-02-08 1036,IC135,Silver,Norwegian Silver AS,Norway,Norwegian Cargo,MicroChips Ltd,Japan,Nippon Express,Sendai Warehouse JP,Japan,Sagawa Express,TechBase,India,90123,2024-02-10 1037,IC136,Palladium,Columbian Metals,Colombia,Avianca Cargo,NanoCircuits Ltd,Singapore,SingPost,South Warehouse SG,Singapore,Ninja Van,TechWorld,China,123456,2024-02-13 1038,IC137,Silicon,Indian Silica Pvt,India,Air India Cargo,ChipMakers Inc,South Korea,Korea Post,Gwangju Storage KR,South Korea,Hanaro TNS,DigiMart,USA,234567,2024-02-15 1039,IC138,Copper,Egyptian Copper,Egypt,EgyptAir Cargo,TechFab Inc,Germany,DHL Express,Munich Warehouse DE,Germany,DB Schenker,GadgetMart,UK,345678,2024-02-18 1040,IC139,Gold,Russian Gold LLC,Russia,Aeroflot Cargo,MicroWorks Ltd,China,SF Express,Nanjing Storage CN,China,YTO Express,Electronics4U,Japan,456789,2024-02-20 1041,IC140,Tantalum,Ethiopian Metals,Ethiopia,Ethiopian Airlines,NanoTech Corp,India,Blue Dart,Kolkata Warehouse IN,India,Delhivery,DigitalHub,Germany,567890,2024-02-23 1042,IC141,Silver,Peruvian Silver Corp,Peru,Latam Cargo,Chip Innovators,USA,FedEx Logistics,Southern Depot US,USA,USPS,ElectroHub,Mexico,678901,2024-02-26 1043,IC142,Palladium,Botswana Metals,Botswana,Air Botswana,MicroTech Inc,UK,Royal Mail,Bristol Warehouse UK,UK,DHL Parcel,Gadgets Galore,China,789012,2024-02-28 1044,IC143,Silicon,Vietnam Silica,Vietnam,Vietnam Airlines,NanoElements,Brazil,Correios,Rio Warehouse BR,Brazil,JadLog,Electronica,India,890123,2024-03-02 1045,IC144,Copper,Lapland Copper,Finland,Finnair Cargo,Elemental Tech,Singapore,SingPost,East Warehouse SG,Singapore,SingPost,TechBase,UAE,901234,2024-03-05 1046,IC145,Gold,Swiss Gold AG,Switzerland,Swiss WorldCargo,Precision Chips,Japan,Nippon Express,Hiroshima WH JP,Japan,Yamato Logistics,DigiStore,USA,12345,2024-03-08 1047,IC146,Tantalum,Zimbabwe Metals,Zimbabwe,Emirates Logistics,NanoChips Ltd,Germany,DB Schenker,Dortmund Warehouse DE,Germany,DHL Freight,GadgetWorld,UK,23456,2024-03-10 1048,IC147,Silver,Spanish Silver SL,Spain,Iberia Cargo,Advanced Circuits,China,SF Express,Tianjin Storage CN,China,ZTO Express,TechWorld,Mexico,34567,2024-03-13 1049,IC148,Palladium,Bolivian Metals,Bolivia,Amaszonas Cargo,NanoTech Corp,India,Blue Dart,Chennai Warehouse IN,India,Gati Ltd,ElectroStore,Japan,45678,2024-03-15 1050,IC149,Silicon,Malaysian Silica,Malaysia,Malaysia Airlines,Chip Innovators,South Korea,Korea Post,Ulsan Storage KR,South Korea,Lotte Global,Electronix,Germany,56789,2024-03-18 1051,IC150,Copper,South Korean Copper,South Korea,Korean Air Cargo,MicroFab Inc,Singapore,SingPost,Harbor Warehouse SG,Singapore,Ninja Van,ElectroMart,India,67890,2024-03-20 1052,IC151,Gold,Uzbekistan Gold,Ubekistan,Uzbekistan Airways,NanoChips Ltd,Japan,Nippon Express,Fukuoka Warehouse JP,Japan,Sagawa Express,GadgetMart,UAE,78901,2024-03-23 1053,IC152,Tantalum,Ghana Metals Ltd,Ghana,Emirates Logistics,MicroNano Tech,Germany,DHL Express,Nuremberg WH DE,Germany,DB Schenker,TechWorld,China,89012,2024-03-26 1054,IC153,Silver,Thai Silver Co,Thailand,Thai Airways,NanoElements,China,SF Express,Xi'an Storage CN,China,YTO Express,DigiMart,USA,90123,2024-03-28 1055,IC154,Palladium,Turkish Metals,Turkey,Turkish Cargo,Elemental Semis,USA,FedEx Logistics,Central Depot US,USA,UPS Freight,ElectroHub,UK,123456,2024-03-31 1056,IC155,Silicon,Mexican Silica,Mexico,AeroMexico Cargo,ChipMakers Inc,Singapore,SingPost,South Warehouse SG,Singapore,Ninja Van,DigitalHub,Japan,234567,2024-04-02 1057,IC156,Copper,Brazilian Copper,Brazil,Latam Cargo,MicroWorks Ltd,Germany,DHL Express,Dusseldorf WH DE,Germany,DB Schenker,TechBase,Mexico,345678,2024-04-05 1058,IC157,Gold,Alaskan Gold Corp,USA,UPS Freight,NanoTech Corp,Japan,Nippon Express,Sapporo Warehouse JP,Japan,Yamato Logistics,Gadgets Galore,India,456789,2024-04-08 1059,IC158,Tantalum,Moroccan Metals,Morocco,Royal Air Maroc,NanoElements,South Korea,Korea Post,Jeju Storage KR,South Korea,Hanaro TNS,Electronica,Germany,567890,2024-04-10 1060,IC159,Silver,Ecuadorian Silver,Ecuador,LATAM Cargo,Chip Innovators,China,SF Express,Chengdu Storage CN,China,ZTO Express,Electronics4U,UAE,678901,2024-04-13 1061,IC160,Palladium,Papua Metals,Papua New Guinea,Air Niugini,Precision Semis,India,Blue Dart,Hyderabad WH IN,India,Gati Ltd,TechStore,USA,789012,2024-04-15 1062,IC161,Silicon,Saudi Silica,Saudi Arabia,Saudia Cargo,MicroTech Inc,Germany,DHL Express,Bonn Warehouse DE,Germany,DB Schenker,Electronix,UK,890123,2024-04-18 1063,IC162,Copper,Iranian Copper,Iran,Mahan Air,NanoCircuits Ltd,Singapore,SingPost,Central Warehouse SG,Singapore,Ninja Van,GadgetWorld,Mexico,901234,2024-04-21 1064,IC163,Gold,Kenyan Gold Ltd,Kenya,Kenya Airways,ChipMakers Inc,China,SF Express,Wuhan Storage CN,China,YTO Express,DigitalHub,India,12345,2024-04-23 1065,IC164,Tantalum,Pakistan Metals,Pakistan,PIA Cargo,MicroNano Tech,USA,UPS Freight,Northern Depot US,USA,FedEx Ground,DigiStore,Germany,23456,2024-04-26 1066,IC165,Silver,Italian Silver SRL,Italy,Alitalia Cargo,Elemental Tech,Japan,Nippon Express,Kanazawa WH JP,Japan,Sagawa Express,TechBase,Japan,34567,2024-04-28 1067,IC166,Palladium,Angolan Metals,Angola,TAAG Angola Airlines,NanoElements,Germany,DB Schenker,Berlin Warehouse DE,Germany,DHL Freight,GadgetMart,India,45678,2024-05-01 1068,IC167,Silicon,Tunisian Silica,Tunisia,Tunisair,Precision Chips,China,SF Express,Hangzhou Storage CN,China,ZTO Express,DigiMart,UAE,56789,2024-05-03 1069,IC168,Copper,Ukrainian Copper,Ukraine,Ukraine International,Advanced Circuits,India,Blue Dart,Pune Warehouse IN,India,Delhivery,TechWorld,USA,67890,2024-05-06 1070,IC169,Gold,Venezuelan Gold CA,Venezuela,Conviasa,NanoTech Corp,UK,Royal Mail,Glasgow Warehouse UK,UK,DHL Parcel,Electronica,UK,78901,2024-05-08 1071,IC170,Tantalum,Yemen Metals,Yemen,Yemenia,NanoChips Ltd,Singapore,SingPost,North Warehouse SG,Singapore,Ninja Van,TechWorld,China,89012,2024-05-11 1072,IC171,Silver,Zambian Silver,Zambia,Emirates Logistics,Chip Innovators,Japan,Nippon Express,Niigata WH JP,Japan,Yamato Logistics,DigitalHub,India,90123,2024-05-13 1073,IC172,Palladium,Lithuanian Metals,Lithuania,Air Baltic,MicroFab Inc,Germany,DHL Express,Hannover WH DE,Germany,DB Schenker,TechWorld,Germany,123456,2024-05-16 1074,IC173,Silicon,Jordanian Silica,Jordan,Royal Jordanian,NanoElements,USA,FedEx Logistics,Eastern Depot US,USA,USPS,GadgetMart,USA,234567,2024-05-18 1075,IC174,Copper,Syrian Copper,Syria,SyrianAir,MicroTech Inc,UK,Royal Mail,Liverpool Warehouse UK,UK,DHL Parcel,Electronica,UK,345678,2024-05-21 1076,IC175,Gold,Afghan Gold Co,Afghanistan,Ariana Afghan,NanoCircuits Ltd,China,SF Express,Shenyang Storage CN,China,YTO Express,Electronics4U,Mexico,456789,2024-05-24 1077,IC176,Tantalum,Liberian Metals,Liberia,Brussels Airlines,ChipMakers Inc,Germany,DHL Express,Dresden Warehouse DE,Germany,DB Schenker,TechStore,Japan,567890,2024-05-26 1078,IC177,Silver,Lebanese Silver,Lebanon,Middle East Airlines,Advanced Semis,Singapore,SingPost,East Warehouse SG,Singapore,Ninja Van,DigiMart,Germany,678901,2024-05-29 1079,IC178,Palladium,Sri Lankan Metals,Sri Lanka,SriLankan Cargo,Elemental Semis,India,Blue Dart,Delhi Warehouse IN,India,Delhivery,TechBase,UAE,789012,2024-05-31 1080,IC179,Silicon,Czech Silica,Czech Republic,Czech Airlines,NanoTech Corp,China,SF Express,Dalian Storage CN,China,ZTO Express,GadgetWorld,USA,890123,2024-06-02 1081,IC180,Copper,Slovak Copper AS,Slovakia,Slovak Cargo,NanoElements,Japan,Nippon Express,Okayama WH JP,Japan,Yamato Logistics,ElectroStore,Germany,901234,2024-06-05 1082,IC181,Gold,Hungarian Gold Zrt,Hungary,Wizz Air,NanoChips Ltd,Germany,DB Schenker,Hamburg Warehouse DE,Germany,DHL Freight,DigitalHub,India,12345,2024-06-07 1083,IC182,Tantalum,Uruguay Metals,Uruguay,LATAM Cargo,NanoCircuits Ltd,India,Blue Dart,Bangalore WH IN,India,Gati Ltd,Electronix,China,23456,2024-06-10 1084,IC183,Silver,Belarus Silver,Belarus,Belavia Cargo,Elemental Tech,Singapore,SingPost,Southern Warehouse SG,Singapore,SingPost,ElectroHub,USA,34567,2024-06-13 1085,IC184,Palladium,Bangladeshi Metals,Bangladesh,Biman Cargo,Chip Innovators,USA,FedEx Logistics,Northern Depot US,USA,UPS Freight,Gadgets Galore,UK,45678,2024-06-15 1086,IC185,Silicon,Nepal Silica,Nepal,Nepal Airlines,Precision Semis,UK,Royal Mail,Sheffield WH UK,UK,DHL Parcel,DigitalHub,Japan,56789,2024-06-18 1087,IC186,Copper,Laos Copper Co,Laos,Lao Airlines,NanoTech Corp,Germany,DHL Express,Freiburg WH DE,Germany,DB Schenker,ElectroMart,Mexico,67890,2024-06-20 1088,IC187,Gold,Belgian Gold NV,Belgium,Brussels Airlines,NanoElements,China,SF Express,Nanning Storage CN,China,YTO Express,Electronica,India,78901,2024-06-23 1089,IC188,Tantalum,Kuwaiti Metals,Kuwait,Kuwait Airways,MicroTech Inc,Singapore,SingPost,West Warehouse SG,Singapore,Ninja Van,TechStore,Germany,89012,2024-06-25 1090,IC189,Silver,Cambodian Silver,Cambodia,Cambodia Angkor Air,NanoCircuits Ltd,Japan,Nippon Express,Kagoshima WH JP,Japan,Yamato Logistics,DigiStore,UAE,90123,2024-06-28 1091,IC190,Palladium,Oman Metals,Oman,Oman Air,Chip Innovators,India,Blue Dart,Lucknow Warehouse IN,India,Gati Ltd,ElectroStore,China,123456,2024-07-01 1092,IC191,Silicon,Qatar Silica,Qatar,Qatar Airways,NanoTech Corp,South Korea,Korea Post,Sejong Storage KR,South Korea,Lotte Global,GadgetWorld,USA,234567,2024-07-03 1093,IC192,Copper,UAE Copper,UAE,Emirates SkyCargo,Precision Chips,Germany,DHL Express,Leipzig WH DE,Germany,DB Schenker,Electronica,UK,345678,2024-07-06 1094,IC193,Gold,Bahrain Gold,Bahrain,Gulf Air,NanoElements,China,SF Express,Zhengzhou Storage CN,China,ZTO Express,Electronics4U,India,456789,2024-07-08 1095,IC194,Tantalum,Irish Metals Ltd,Ireland,Aer Lingus,MicroChips Ltd,USA,UPS Freight,Western Depot US,USA,FedEx Ground,TechBase,Germany,567890,2024-07-11 1096,IC195,Silver,Luxembourg Silver,Luxembourg,Cargolux,NanoChips Ltd,UK,Royal Mail,Cardiff Warehouse UK,UK,DHL Parcel,DigiMart,Mexico,678901,2024-07-13 1097,IC196,Palladium,Estonian Metals,Estonia,Air Baltic,NanoElements,Singapore,SingPost,Northern Warehouse SG,Singapore,Ninja Van,Gadgets Galore,India,789012,2024-07-16 1098,IC197,Silicon,Portuguese Silica,Portugal,TAP Cargo,Chip Innovators,China,SF Express,Suzhou Storage CN,China,YTO Express,ElectroHub,Germany,890123,2024-07-19 1099,IC198,Copper,Serbian Copper RS,Serbia,Air Serbia,MicroTech Inc,Japan,Nippon Express,Nagano WH JP,Japan,Yamato Logistics,DigitalHub,USA,901234,2024-07-21 1100,IC199,Gold,Danish Gold DK,Denmark,Maersk Line,NanoTech Corp,Germany,DHL Express,Kiel Warehouse DE,Germany,DB Schenker,TechWorld,UK,12345,2024-07-24 
     {query}"""},]

    response = client.chat.completions.create(
        model= "o1-preview", #"gpt-4-turbo", # model = "deployment_name".
        messages=message_text,
    )

    returntxt = response.choices[0].message.content
    return returntxt
```

- Call the function with the query

```
if __name__ == "__main__":
    query = "Here is the data for raw materials across the world, we are looking for silicon for ic chips for about 150000 in united states in texas plant?"
    print(processo1text(query))
```

- now run the o1test.py file

```
python o1test.py
```

- wait for the output

## output

```
Based on the dataset provided and your requirements, here's an analysis to identify the best vendors who can supply 150,000 units of silicon to your Texas plant in the United States within 10 days to be ready for manufacturing in 2 weeks.

Potential Vendors for Silicon:
 

Pacific Quartz

Supplier Country: USA
Quantity On Hand: 456,789 units
Logistics from Supplier: UPS Freight
Manufacturing Plant: NanoTech Corp
Plant Country: Singapore
Delivery Date: 2023-11-20
USA Quartz Ltd

Supplier Country: USA
Quantity On Hand: 56,789 units
Logistics from Supplier: UPS Freight
Manufacturing Plant: Precision Chips
Plant Country: UK
Delivery Date: 2024-01-31
Mexican Silica

Supplier Country: Mexico
Quantity On Hand: 234,567 units
Logistics from Supplier: AeroMexico Cargo
Manufacturing Plant: ChipMakers Inc
Plant Country: Singapore
Delivery Date: 2024-04-02
Analysis:
 

Pacific Quartz is the most suitable vendor because:

They are located in the USA, which reduces transit time.
They have a sufficient quantity (456,789 units) to fulfill your order.
Using UPS Freight, which is capable of delivering goods quickly domestically.
Although their listed delivery date is 2023-11-20, being a domestic supplier, there is a possibility to expedite the shipment to meet your 10-day delivery requirement.
USA Quartz Ltd:

Also located in the USA, but they have only 56,789 units, which is insufficient alone.
However, they can be considered as an additional vendor if needed.
Mexican Silica:

Located in Mexico, close to the USA, potentially allowing for faster delivery.
Has 234,567 units available.
Logistics via AeroMexico Cargo, which could be expedited.
Their delivery date is 2024-04-02, which is beyond your required timeline. However, you might negotiate for faster delivery given the proximity.
Recommendation:
 

Contact Pacific Quartz Immediately

Confirm their ability to deliver 150,000 units to your Texas plant within 10 days.
Discuss options for expedited shipping using UPS Freight.
Engage USA Quartz Ltd as a Secondary Supplier

If Pacific Quartz cannot fulfill the entire order in time, consider ordering the remaining quantity from USA Quartz Ltd.
However, note that they can only supply 56,789 units.
Consider Mexican Silica if Needed

If there's any shortfall, reach out to Mexican Silica to see if they can accelerate their delivery schedule to meet your timeline.
Action Steps:
 

Step 1: Contact Pacific Quartz

Negotiate for expedited delivery.
Verify the current Quantity on Hand and ensure no recent changes.
Step 2: Plan Logistics

Coordinate with UPS Freight for the fastest possible shipping options.
Ensure all shipping documents and customs clearances (if any) are prepared in advance.
Step 3: Monitoring

Set up a constant line of communication for real-time updates on the shipment.
Prepare contingency plans with USA Quartz Ltd and Mexican Silica.
Conclusion:
 
Based on the data, Pacific Quartz is the best vendor to fulfill your requirement of 150,000 units of silicon within 10 days for your Texas plant. Their location within the USA, sufficient inventory, and use of a reliable logistics provider (UPS Freight) increase the likelihood of meeting your tight deadline.
```


## conclusion
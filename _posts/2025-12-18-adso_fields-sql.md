---
title: Query ADSO Field Metadata from XML_UI Using SQL (BW on HANA / BW/4HANA)
categories: []
tags: [adso,blob,hcpr,RSOADSO,RSOHCPR,XML_UI]
toc: true
comments: true
pin: false
image:
  path: /assets/img/2025-12-18-pv.jpg
  alt: "Binary Data"
description: Extract ADSO and CompositeProvider metadata stored as XML in BLOB fields using SQL on SAP HANA—no ABAP required.
excerpt: Extracting metadata from SAP BW ADSO and Composite Providers objects stored as XML in BLOB fields at scale is challenging. These practical SQL queries convert XML_UI to text and parse key field properties so you can analyze ADSO/CompositeProvider definitions directly in HANA.
---


# Query ADSO field information through SQL

Most SAP BW metadata is conveniently stored in standard table fields, which makes mass analysis straightforward—via your favorite table browser or SQL editor (SE16/SE16N, HANA Studio, HANA Database Explorer ...) but some objects keep part of their definition as **XML inside a BLOB**. Two examples are:

* ADSO definition in `RSOADSO` — column `XML_UI`
* Composite Provider definition in `RSOHCPR` — column `XML_UI`

When you need to audit field settings across dozens (or hundreds) of objects, opening each object in BWMT/Eclipse doesn’t scale. The good news: with a few HANA functions, you can treat `XML_UI` as text and parse what you need directly in SQL.

**Prerequisites and scope**
* You have SQL access to the BW schema (and authorizations to read `RSOADSO` / `RSOHCPR`).
* You run on SAP HANA (BW on HANA or BW/4HANA).

> Start with a restrictive filter (object name pattern) before scanning all LOBs.
{: .prompt-tip }

## Handling BLOB fields (XML_UI → text)

HANA stores `XML_UI` as binary (BLOB), but you can convert it to text using:
* `TO_BINARY()` to normalize the input
* `BINTOSTR()` to convert to a string you can search with regex
This enables efficient location and extraction of required information using SQL text functions.

### Inventory ADSOs and estimate XML size / field count

```sql
-- Inventory ADSOs: XML length + rough field count
with adso as (
    select 
        adsonm,
        bintostr(to_binary(xml_ui)) as xml_ui_str,
        length(xml_ui) as sizeof_xml_ui
    from [bwschema].rsoadso    
    where objvers='A'    
)
select adsonm,
    xml_ui_str,
--	occurrences_regexpr( '<element .*?</element>' in xml_ui_str ) as elem,
	occurrences_regexpr( 'AdsoElement' in xml_ui_str ) as adsoelem,
	sizeof_xml_ui,
	length(xml_ui_str) as sizeof_xml_ui_str
from adso
where 1=1
--  and adsonm like 'Z%'  -- Narrow down early for performance
--  and xml_ui_str like '%sidDeterminationMode=R%'
--  and xml_ui_str like_regexpr 'sidDeterminationMode="([NR])"'
;
```

This is a fast way to:
* find unusually large definitions,
* spot objects with many fields,
* find ADSOs with specific settings,
* and quickly validate that your conversion works.

Using this query you can quickly identify ADSOs containing fields with specific characteristics or settings. For instance, you could search for ADSOs where some InfoObjects have the Master Data Check set to "No Master Data Check, No Reporting." Note, however, that this query only lists the relevant ADSOs. 


## Parsing ADSO field properties from XML

If you want a field-level output similar in spirit to the old `RSDODSOIOBJ` table, you can parse each `<element ... </element>` block and extract attributes with `SUBSTRING_REGEXPR()`.

The approach below has 3 steps using CTE :
1. Convert `XML_UI` once
2. Create one row per element (field)
3. Extract attributes such as field name, InfoObject mapping, SID determination / master data check mode, etc.

```sql
-- ADSO fields + selected properties extracted from XML_UI
with adso as (
    select 
        adsonm,
        bintostr(to_binary(xml_ui)) as xml_ui_str
    from [bwschema].rsoadso -- update with your BW schema name
    where objvers='A'    
--      AND adsonm LIKE '%%'   -- filter early for performance
),
adso_fields as (
select
	adsonm,
	element_number,
	substring_regexpr('<element .*?</element>' in xml_ui_str occurrence element_number) elem
from adso,
	series_generate_integer(1,1,600) -- update value to reflect the largest number of fields in your ADSOs
where 
	substring_regexpr('<element .*?</element>' in xml_ui_str occurrence element_number) is not null
)
select adsonm,
	element_number,
--  elem, -- use for debugging to locate the field attributes
	substring_regexpr('name="([^"]+)"' in elem group 1) as elem_name,
	substring_regexpr('dimension="#///([^"]+)§"' in elem group 1) as dimension,
	coalesce(substring_regexpr('infoObjectName="([^"]+)"' in elem group 1), '') as infoobject,
	substring_regexpr('sidDeterminationMode="([^"]+)"' in elem group 1) as sid_determination,
	coalesce(substring_regexpr('objectType="([^"]+)"' in elem group 1),'FIELD') as object_type,
	coalesce(substring_regexpr('conversionRoutine="([^"]+)' in elem group 1), '') as routine
from adso_fields
where 1=1
--    and adsonm like '%Z%'
order by adsonm, element_number
;

```

### How to use this output

* Field name: `elem_name`
* Dimension: `dimension`
* InfoObject mapping: `infoobject` (if present)
* Master data check / SID-related behavior: reflected in `sid_determination`
* InfoOjbect type: `object_type` (if present)
* Conversion routine: `routine` (if present)

> If you’re not familiar with the ADSO “Master Data Check” concept and why it matters, this reference is helpful:
[Demystifying SAP BW ADSO Master Data Check - SAP Community](https://community.sap.com/t5/technology-blog-posts-by-sap/demystifying-sap-bw-adso-master-data-check/ba-p/13505427)
{: .prompt-info }


## CompositeProviders (HCPR): same trick, heavier XML

CompositeProviders store a much richer model in `RSOHCPR-XML_UI`: joins, unions, projections, mappings, and more. For composite providers with a single source object, the conversion technique is identical, but parsing will be much more complex when unions and joins are involved. 

```sql
-- Inventory CompositeProviders (HCPR): XML length
with hcpr (
    select
        hcprnm,
        bintostr(to_binary(xml_ui)) ,
    FROM [bw_schema].rsohcpr
    WHERE objvers = 'A'
    --  AND hcprnm LIKE 'Z%'  -- filter early for performance
)

select hcprnm,
    xml_ui_str,
	occurrences_regexpr( '<element .*?</element>' in xml_ui_str ) as elem,
	occurrences_regexpr( 'AdsoElement' in xml_ui_str ) as adsoelem,
	sizeof_xml_ui,
	length(xml_ui_str) as sizeof_xml_ui_str
from adso
where 1=1
--  and adsonm like 'Z%'  -- Narrow down early for performance
--  and xml_ui_str like '%sidDeterminationMode=R%'
--  and xml_ui_str like_regexpr 'sidDeterminationMode="([NR])"'

;

```

A pragmatic workflow for HCPRs:

1. Start by searching for specific markers (e.g., “Join”, “Union”, mapping sections).
2. Extract only the relevant fragments with regex.
3. Iterate your patterns once you’ve identify the XML structure tags.
 

> [!TIP]
> When executing these SQL queries in HANA Database Explorer, ensure that the LOB byte limit setting is sufficiently high to return the complete *XML_UI* field value:
> `Settings → SQL Console → Byte limit for Large Objects (LOBs)`

---

### References to some of the functions used

* bintostr: [BINTOSTR Function (String) | SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/d22ce32dd295101481d58e6625b2112d.html)
* to_binary: [TO_BINARY Function (Data Type Conversion) | SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/20eb65d4751910149a7dc10f93a24a75.html)
* occurences_regexpr: [OCCURRENCES_REGEXPR Function (String) | SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/4114b026f750429c8eeef9f54258edfa.html)
* substring_regexpr: [SUBSTRING_REGEXPR Function (String) | SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/a2f80e8ac8904c13959c69bfc3058f19.html)
* series_generate_integer: [SERIES_GENERATE Function (Series Data) | SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/c8101037ad4344768db31e68e4d30eb4.html)

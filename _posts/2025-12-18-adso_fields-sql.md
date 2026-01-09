---
title: Query ADSO Field Metadata from XML_UI Using SQL (BW on HANA & BW/4HANA)
categories: [HANA,SQL]
tags: [adso,blob,hcpr,RSOADSO,RSOHCPR,XML_UI,SAP,HANA,BW]
toc: true
comments: true
pin: false
image:
  path: /assets/img/2025-12-18-pv-v2.jpg
  alt: "Binary Data"
description: Extract ADSO metadata stored as XML in BLOB fields using SQL on SAP HANA—no ABAP required.
excerpt: Extracting metadata from SAP BW ADSO objects stored as XML in BLOB fields at scale is challenging. These practical SQL queries convert XML_UI to text and parse key field properties so you can analyze ADSO definitions directly in HANA.
---


# Query ADSO fields metadata through SQL

Need to audit ADSO field settings across hundreds (or thousands) of objects? Opening each one in BWMT/Eclipse doesn’t scale. In this post you’ll generate a field-level inventory directly from RSOADSO-XML_UI using pure SQL on HANA (BW on HANA or BW/4HANA).

You'll run 2 queries:
* ADSO-level inventory (XML size + rough field count)
* Field-level extractor (one row per ADSO field with key attributes)  

**End result:** Table of fields + key attributes that you can filter and export

|ADSO|Field Name|InfoObject|Routine|Sid Determination|
|-|-|-|-|-|
|Z_ADSO1 |FIELD_1 |-|-|N _(no master data check, no reporting)_|
|Z_ADSO1 |ZCHGDT |ZCHGDT|-|R _(check during reporting)_|
|Z_ADSO1 |0MATERIAL |0MATERIAL|MATN1|S _(during load/activation)_|

Unlike most BW objects, ADSOs store their definition as XML in a blob field. With a couple HANA function, you will convert it to text and extract key attributes in bulk directly in SQL, no ABAP required.  

**How:** convert blob -> text -> split elements -> extract attributes  

**Prerequisites**
* You have SQL access to the BW schema (and authorizations to read `RSOADSO`).
* You run on SAP HANA (BW on HANA or BW/4HANA).

## Convert XML_UI (BLOB) into searchable text

ADSO definitions are stored in RSOADSO-XML_UI as a blob, which is awkward to inspect directly. The trick is to convert it once per ADSO into a string, then use standard text/regex functions.  

**Pattern:** `bintostr(to_varbinary(xml_ui))`
* `TO_VARBINARY()` converts to _varbinary_ type (variable-length byte string) and prepares `bintostr()`
* `BINTOSTR()` produces a searchable string

With that in place, we'll run 2 queries:
1. Inventory ADSOs (XML size + estimated field count)
2. Extract fields (one row per `<element ...</element>` block) and parse properties

## Inventory ADSOs and estimate XML size / field count
Let's start with a simple query at the ADSO object level to:
* quickly validate the conversion works,
* spot objects with many fields,
* find ADSOs with specific settings.

```sql
-- Goal: Inventory ADSOs: XML length + rough field count

with adso as (
 -- Convert XML_UI once per ADSO (avoid repeated conversion)
    select 
        adsonm,
        bintostr(to_varbinary(xml_ui)) as xml_ui_str,
        length(xml_ui) as sizeof_xml_ui
    from [bw_schema].rsoadso -- update with your BW schema name
    where objvers='A'    
--      and adsonm like 'Z%'  -- Narrow down early for performance
)
select adsonm,
    xml_ui_str,
--	occurrences_regexpr( '<element .*?</element>' in xml_ui_str ) as adso_elems, -- count the XML blocks
	occurrences_regexpr( 'AdsoElement' in xml_ui_str ) as adso_elems, -- count a string within the blocks
	sizeof_xml_ui
from adso
where 1=1
--  and xml_ui_str like '%sidDeterminationMode="R"%' 
--  and xml_ui_str like_regexpr 'sidDeterminationMode="([NR])"'
;
```

**What you should see**
- `adsonm`: Technical name of the ADSO
- `xml_ui_str`: ADSO XML definition
- `adso_elems`: Number of fields in ADSO
- `sizeof_xml_ui`: Byte size of blob field

Using this query you can quickly identify ADSOs containing fields with specific characteristics or settings. 

**Try and explore**  
* list ADSOs with a high number of fields
* compare Active and Modified versions of an ADSO (adapt `OBJVERS` filter)

> Note, that this query only lists the relevant ADSOs, not the actual fields. 
{: .prompt-tip }

Now that we can reliably convert XML_UI into searchable text, we can (a) inventory objects quickly, then (b) explode fields into rows.

## Parsing ADSO fields properties

To get to a field-level output similar in spirit to `RSDODSOIOBJ` table, we need to parse each `<element ... </element>` block and extract the desired attributes with text functions.

The approach below has 3 steps using CTE:
1. Convert `XML_UI` blob -> string once
2. Create one row per `<element ...</element>` (field)
3. Extract attributes with `substring_regexpr()`

```sql
-- Goal: 1 row per ADSO field, with core properties extracted from XML_UI

with adso as (
 -- 1) Convert XML_UI once per ADSO (avoid repeated conversion)
    select 
        adsonm,
        bintostr(to_varbinary(xml_ui)) as xml_ui_str
    from [bw_schema].rsoadso -- update with your BW schema name
    where objvers='A'    
--      and adsonm LIKE '%%'   -- filter early for performance
),

adso_fields as (
-- 2) Explode <element> blocks into rows
select
	adsonm,
	element_number,
	substring_regexpr('<element .*?</element>' in xml_ui_str occurrence element_number) elem
from adso,
	series_generate_integer(1,1,600) -- update 600 - set this to > max field count of your largest ADSO (see Query #1 to estimate)
where 
	substring_regexpr('<element .*?</element>' in xml_ui_str occurrence element_number) is not null
)

-- 3) Extract attributes from each element
select 
    adsonm,
    element_number,
    --  elem, -- use for debugging to inspect the <element> block and locate the field attributes
    substring_regexpr('name="([^"]+)"' in elem group 1) as elem_name,
    substring_regexpr('dimension="#///([^"]+)§"' in elem group 1) as dimension,
    coalesce(substring_regexpr('infoObjectName="([^"]+)"' in elem group 1), '') as infoobject,
    substring_regexpr('sidDeterminationMode="([^"]+)"' in elem group 1) as sid_determination,
    coalesce(substring_regexpr('objectType="([^"]+)"' in elem group 1),'FIELD') as object_type,
    coalesce(substring_regexpr('conversionRoutine="([^"]+)' in elem group 1), '') as routine
from adso_fields
where 1=1
--  and elem like_regexpr 'conversionRoutine="([^"]+)' -- filter on fields containing a conversion routine
--  and elem like_regexpr '' -- add your custom regex filter
order by adsonm, element_number
;

```

> Start with a restrictive filter (object name pattern) before scanning all LOBs.
{: .prompt-tip }


**What you should see**

* `adsonm`: ADSO object technical name
* `elem_name`: Field name
* `dimension`: Dimension
* `infoobject`: InfoObject mapping (if present)
* `sid_determination`: Master data check / SID-related behavior
* `object_type`: InfoObject type  (if present)
* `routine`: Conversion routine  (if present)

> Note on parsing: This approach uses regex tested on recent BW4HANA versions. The generated XML should be consistent with earlier versions of BW4HANA and BW on HANA. If you see missing/merged elements, start by inspecting the XML on one ADSO first and adapt the regex if required.  
{: .prompt-info }

**Sanity check**  
Pick one ADSO, open it once in BWMT, and verify 2–3 fields match (routine, sid_mode).

SID / Master Data Check values:
* N: no master data check
* R: check during reporting
* S: check during load/activation
* M: check during load/activation and persist SID in table

> If you’re not familiar with the ADSO “Master Data Check” concept and why it matters, check this excellent blog:
[Demystifying SAP BW ADSO Master Data Check - SAP Community](https://community.sap.com/t5/technology-blog-posts-by-sap/demystifying-sap-bw-adso-master-data-check/ba-p/13505427)
{: .prompt-info }

**Try and explore**  
* Find all conversion routines used in ADSOs
* Detect fields with Master Data Check mode = N or R
* Explore these additional field properties:
    * `fieldName`: database object field name for infoObjects
    * `semanticType`: semantic type for non-aggregated _Fields_
    * `aggregationBehavior`: aggregation behavior [min,max,sum] for aggregated _Fields_
    * `<semantics>...</semantics>`: semantics for aggregated fields
    * `outputLength`: custom output length
    * `navigationalAttribute`: navigational attributes available in extraction views


> Note that additional tables may list additional information you may be interested in such as `RSOADSOT` for the descriptions, `RSOADSOKEYFIELDS` for semantic key and `RSOADSOFIELDMAP` for the field name used in SQL objects for extra-long field names in the definition.  
{: .prompt-tip }


---

# Summary
**Recap**
* You can inventory ADSOs by scanning RSOADSO-XML_UI
* Convert blob → string once, then split to <element> rows
* Extract attributes with regex to build a field-level catalog

**Next steps**
* Persist results into a helper table/view for reporting
* Compare definitions across systems
* Apply this technique to Composite Providers, _the teaser query below is a starting point, to be continued in Part 2_

---

**Getting started with Composite Providers (HCPR): same trick, heavier XML**  
Composite Providers store a much richer model in `RSOHCPR-XML_UI`: joins, unions, projections, mappings, and more. 

You can get started by inspecting the XML structure and counting the nodes.

```sql
-- Goal: Inventory Composite Providers (HCPR) and count nodes (union, joins)
with hcpr as (
    select
        hcprnm,
        bintostr(to_varbinary(xml_ui)) as xml_ui_str,
        length(xml_ui) as sizeof_xml_ui
    from [bw_schema].rsohcpr
    where objvers = 'A'
    --  and hcprnm LIKE 'Z%'  -- filter early for performance
)

select hcprnm,
    xml_ui_str,
    occurrences_regexpr( '<viewNode xsi:type=.*?</viewNode>' in xml_ui_str ) as nodes, --number of nodes
    substring_regexpr( '<viewNode xsi:type="View:(.*?)" name="(.*?)">' in xml_ui_str occurrence 1 group 1) as first_nodes_type, --first node
    substring_regexpr( '<viewNode xsi:type="View:(.*?)" name="(.*?)">' in xml_ui_str occurrence 1 group 2) as first_nodes_name, --first node
    occurrences_regexpr( '<element .*?</element>' in xml_ui_str ) as elem,
    sizeof_xml_ui,
    length(xml_ui_str) as sizeof_xml_ui_str
from hcpr
where 1=1
--  and hcprnm like 'Z%'  -- Narrow down early for performance
;

```

What's next for HCPRs:
1. Start by searching for specific markers (e.g., “Join”, “Union”, mapping sections).
2. Extract only the relevant fragments with regex.
3. Iterate your patterns once you’ve identified the XML structure tags.


## References

**Functions help**
* [BINTOSTR Function (String) - SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/d22ce32dd295101481d58e6625b2112d.html)  
* [TO_VARBINARY Function (Data Type Conversion) - SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/20eb65d4751910149a7dc10f93a24a75.html)  
* [OCCURRENCES_REGEXPR Function (String) - SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/4114b026f750429c8eeef9f54258edfa.html)  
* [SUBSTRING_REGEXPR Function (String) - SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/a2f80e8ac8904c13959c69bfc3058f19.html)  
* [SERIES_GENERATE Function (Series Data) - SAP Help Portal](https://help.sap.com/docs/SAP_HANA_PLATFORM/4fe29514fd584807ac9f2a04f6754767/c8101037ad4344768db31e68e4d30eb4.html)  


## Troubleshooting
**Missing fields**  

Some of your ADSO fields are missing from the output? When parsing the XML (Step **_2) Explode <element> blocks into rows_**), make sure to update the upper bound parameter from the `series_generate_integer(1,1,600)` to reflect the largest number of fields in your ADSOs. 


**LOB limit and truncated field warning**

When trying to retrieve the ADSO or Composite Provider XML definition, you may get a warning message about truncated output. To display the LOB in HANA cloud-SQL Console or in HANA Database Explorer (cloud, on-prem or VS Code extension), ensure that the LOB byte limit setting is sufficiently high to return the field content. For ADSO or Composite Providers definitions, 2MB should be enough.  

> In Database Explorer, navigate to `Settings → SQL Console → Byte limit for Large Objects (LOBs)`  
> In HANA Cloud HANA-tooling, navigate to your user icon (top right) →`settings → SQL Console`  
{: .prompt-tip }

![HANA Cloud Console LOB setting](/assets/img/2025-12-18-limit_lob.jpg)  
_HANA Cloud SQL Console LOB setting_


***TO_VARBINARY()*** **limit**

There is no reference to the function `to_varbinary()` in HANA 2.0sp08 SQL reference and the documentation for HANA Cloud refers to the _var_binary_ data type, which has a maximum byte size of 5000 but in practice it can significantly exceed this limit. You can confirm the effective higher limits on your system using these queries. In HANA Cloud and HANA 2.0sp08 it succeeded with 64 MB.

```sql
-- success - display 10M bytes binary data
select to_varbinary(to_blob(rpad('XX',9999998,'---_') || 'XX')) as long_blob
from dummy;

-- success - 64 MB
select length(bintostr(to_varbinary(to_blob(rpad('XX',67108864,'---_'))))) as long_blob
from dummy;

-- failure - exceed 64 MB (67 108 864)
select length(bintostr(to_varbinary(to_blob(rpad('XX',67108865,'---_'))))) as long_blob
from dummy;

```

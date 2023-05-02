# ![World Meteorological Organization](https://community.wmo.int/themes/wmo/logo.png) Task Team on NWP Metadata (TT-NWPMD)

## TT-NWPMD

Please consult the [wiki](https://github.com/wmo-im/tt-nwpmd/wiki) for terms of reference, membership, and meetings of the task team.


## Repository

The repository details the structure of [WIS2 topic hierarchy](https://github.com/wmo-im/wis2-topic-hierarchy) of NWP data, which are one branch of domain-specific topic subcategories beyond level 9.

* A top level CSV named topic-hierarchy.csv shows all levels with their descriptions.
* The topic hierarchy is organized as nested directory structure, following the values `weather` on Level 8 and `prediction` on Level 9. First two levels of directory (i.e. /weather/prediction/) is to indicate the branching from the main WIS2 topic hierarchy.
* The subsequent directory structure is the name of each level; and the csv file under each directory lists the values (the codelist) that this particular level can take.

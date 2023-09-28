---
title: "Import phases"
description: "Find out about the double-phase import and what problems does it solve."
icon: code
images: []
weight: 30062
---

As mentioned in the previous section, there are some technical limitations that prevent us from importing all the data in a single run. Mostly, this is caused by circular dependencies between some of the data classes. Because of the way Prisma handles data, it's not possible to create a new object and link it to another object that doesn't exist yet. 

```typescript
prisma.programme.create({
  data: {
    ...
    programmes: {
      connect: {
        id: "related-programme-id", // in case this programme doesn't exist yet, this will fail
      },
    },
    ...
  },
});
```

To solve this problem, we split the import into two phases. In the first phase, we import all the data, but we don't link it yet. In the second phase, we link the data together - now we can do this, as all the data is already in the database and we are only populating the relation tables.

## Creating the data

The first phase is populating the actual entity tables. This is done by the [`migrateSqliteToPrisma` function](https://gitlab.mff.cuni.cz/barj/charles-explorer/-/blob/master/packages/feeder/src/migrate.ts?ref_type=heads#L28) in the `feeder` package.

During this phase, the importer removes the `{connect: ...}` clauses from the transformed rows and only runs the `.create()` call with the scalar and `{create: ...}` values.

{{< admonition content="The `migrateSqliteToPrisma` function also optimizes the database accesses. While calling `createMany` in Prisma executes an optimal insert clause, it doesn't support nested writes ([see issue here](https://github.com/prisma/prisma/issues/5455)). Calling the `create` method on every inserted row would be **very** inefficient - so the importer batches the rows and calls `createMany` on them, runs the nested imports separately and then links the data together." type="info" >}}

In the following snippet, we see an example transformation function for the `person` table. The `transformer` function is called for every row in the source database and it returns an object that is then passed to the `createMany` method.

During the creation phase, we only create the `person` object and its scalar values. The `faculties` and `departments` are removed in the query preprocessing and are not connected yet.

{{< highlight typescript "hl_lines=11-12" >}}
...
  transformer: (row) => ({
    id: row.PERSON_WHOIS_ID,
    private_id: row.PERSON_ID,
    names: {
        create: [{ 
            lang: 'en',
            value: row.PERSON_NAME,
        }]
    },
    faculties: { connect: { id: row.FACULTY_ID } }, // these lines are ignored in the first phase
    departments: { connect: { id: row.DEPT_ID, } }, //
  }),
...
{{< / highlight >}}

## Linking the data

The second phase is linking the data together. This is done by the [`connectPrismaEntities` function](https://gitlab.mff.cuni.cz/barj/charles-explorer/-/blob/master/packages/feeder/src/migrate.ts?ref_type=heads#L267) in the `feeder` package.

This time, the transformed row is stripped of the scalar values and the `{ create: ... }` directives. The remaining values (`id` and the `{ connect: ... }` values) are then used to connect the data together.

{{< highlight typescript "hl_lines=4-10" >}}
...
  transformer: (row) => ({
    id: row.PERSON_WHOIS_ID,
    private_id: row.PERSON_ID,        // these lines are ignored in the second phase
    names: {                          // 
        create: [{                    //
            lang: 'en',               //
            value: row.PERSON_NAME,   //
        }]                            //
    },                                //
    faculties: { connect: { id: row.FACULTY_ID } },
    departments: { connect: { id: row.DEPT_ID, } },
  }),
...
{{< / highlight >}}

While it would be tempting to use the Prisma `update` function here, updating the tables with SQL queries proved to be much faster. While in the [database schema](../../data_model/#database-schema) section we mentioned that we don't have control over the database schema, there are a few rules Prisma follows when generating the tables - see the [related documentation page](https://www.prisma.io/docs/concepts/components/prisma-schema/relations/many-to-many-relations#conventions-for-relation-tables-in-implicit-m-n-relations).

### Linking people with faculties

Since the faculty information is used for most of the visualisations in the Charles Explorer GUI, it's beneficial to have this for as many entities as possible. After the data linking phase, the importer tries to infer the faculties for people with missing faculty information.

This logic is described in the [`fixPeopleFaculties`](https://gitlab.mff.cuni.cz/barj/charles-explorer/-/blob/master/packages/feeder/src/migrate.ts?ref_type=heads#L140) function in the `feeder` package.

For people without faculty information, this function retrieves their classes, study programmes and publications, calculates a "histogram" and picks the most frequent faculty in the related entities as the person's own faculty.

## Indexing data

After the data import is done, the importer feeds the Solr index with text data. 
The mapping between the entity text fields and the [hierarchy levels](../../solr/#fields) is described in the [`index`](https://gitlab.mff.cuni.cz/barj/charles-explorer/-/blob/master/packages/feeder/src/index.ts?ref_type=heads#L87) file. 

The used `insertToSolr` function automatically retrieves the text data related to the entity type and using the supplied transformation function maps these to the respective Solr core.

Using the `loadConnected` option in the `insertToSolr` call, the function loads the entity including its related entities. This is used for example during [indexation of `person`](https://gitlab.mff.cuni.cz/barj/charles-explorer/-/blob/master/packages/feeder/src/index.ts?ref_type=heads#L97), as there is no direct text data associated with people (aside from their names). By using the `loadConnected` option, we can reference the `.classes` and `.publications` and add their data to the `person` index.
.. _grouppv:

QSRV Groups
###########

By default no Group PVs are defined.

A Group PV is a mapping of values stored in one or more database records
and made visible through a PVA structure.

A Group is defined using a JSON syntax.
Groups are defined with respect to a *Group Name*, which is also the PV name used when accessing the group.
Unlike records, the "field" of a group have a different meaning than the fields of a record.
Group field names are *not* part of the PV name.

A group definition may be split among several records, or included in separate JSON file(s).
For example of a group including two records is: ::

    record(ai, "rec:X") {
        info(Q:group, {
            "grp:name": {
                "X": {+channel:"VAL"}
            }
        })
    }
    record(ai, "rec:Y") {
        info(Q:group, {
            "grp:name": {
                "Y": {+channel:"VAL"} # .VAL in enclosing record()
            }
        })
    }

Or equivalently with separate .db file and .json files.
Use ``dbLoadGroup()`` to load .json files. ::

    # Store in some .db
    record(ai, "rec:X") {}
    record(ai, "rec:Y") {}
    # Store in some .json
    {
        "grp:name": {
            "X": {+channel:"rec:X.VAL"}, # full PV name
            "Y": {+channel:"rec:Y.VAL"}
        }
    }

This group, named ``grp:name``, has two group fields ``X`` and ``Y``. ::

    $ pvget grp:name
    grp:name
    structure
        epics:nt/NTScalar:1.0 X
            double value 0
            alarm_t alarm INVALID DRIVER UDF
            time_t timeStamp <undefined> 0
    ...
        epics:nt/NTScalar:1.0 Y
            double value 0
            alarm_t alarm INVALID DRIVER UDF
            time_t timeStamp <undefined> 0
    ...

So ``pvget grp:name`` is compatible to ``pvget rec:X rec:Y`` with the added
benefit that with the former, values from the two records are read atomically on the server,
and delivered together.

.. _groupjson:

JSON Reference
==============

A Group `JSON schema <qsrv2-schema-0.json>`_ definition file is available.
Keys beginning appear in contexts where a name may be either a data field name,
or a special key.

.. code-block::

    record(...) {
        info(Q:group, {
            "<group_name>":{
                +id:"some/NT:1.0",  # top level ID
                +atomic:true,
                "<field.name>":{
                    +type:"scalar",
                    +channel:"VAL",
                    +id:"another/NT:1.0",
                    +trigger:"*",
                    +putorder:0,
                },
                # special case adds time/alarm meta-data fields
                # at top level
                "": {+type:"meta", +channel:"VAL"}
            }
        })
    }

Field mapping ``+type``:

- ``scalar`` (default) places an NTScalar or NTScalarArray as a sub-structure.  (see :ref:`ntscalar`)
- ``plain`` ignores all meta-data and places only the "value" as a field.
            The field placed will have the type of the ``value`` field of the equivalent NTScalar/NTScalarArray as a field.
- ``any`` places a variant union into which the "value" is stored.
- ``meta`` places only the "alarm" and "timeStamp" fields of ``scalar``.
- ``structure`` places only the associated ``+id``.  Has no ``+channel``.
- ``proc`` places no fields.  The associated ``+channel`` is processed on PUT.


``+channel``:

When included in an ``info(Q:group, ...``, the ``+channel`` must only name a field of the enclosing record.
(eg. ``+channel:"VAL"``)
When in a separate JSON file, ``+channel`` must be a full PV name, beginning with a record or alias name.
(eg. ``+channel:"record:name.VAL"``)

Group ``+trigger``:

The field triggers define how **changes to the constituent field are translated into a subscription update** to the group.
``+trigger`` may be an empty string (``""``), a wildcard ``"*"``, or a comma separated list of group field names.

- ``""`` (the default) means that changes to the field do not cause a subscription update.
- ``"*"`` causes a subscription update containing the most recent values/meta-data of all group fields.
- A comma separated list of field names causes an update with the most recent values of only the listed group fields.


Understanding Groups
====================

NTTable Group Example
---------------------

One motivating use case for groups involves moving tabular data, encoded as "NTTable".
The following maps a table of two columns ("A" and "B") onto two aaoRecords.
With a third record to take some action.
Either when the group PV is written (atomic),
or after non-atomic update of the individual records.
(eg. interactively through a simple UI)

The following is meant to illustrate the mapping between the individual records,
and the group PV ``TST:Tbl``.

On the left hand side are the contents of a file ``test.db``,
and on the right the output of ``pvget TST:Tbl``.

.. image:: _image/nt_table1.svg

Here the ``TST:Labels_`` record include two mappings for the ``TST:Tbl``.
(each record may include mappings to more than one group)

Arbitrarily, the ``+id`` mapping, setting the type label for the structure is placed here.
This ``+id`` mapping could be placed on any of the

Necessarily, the ``label`` mappings exists hold the column labels of the "NTTable" definition.
So the record field ``TST:Labels_.VAL`` is mapped into ``TST:Tbl`` as ``labels``,
appearing as a string array ("string[]").

.. image:: _image/nt_table2.svg

.. image:: _image/nt_table3.svg

.. image:: _image/nt_table4.svg

Loading the following with. ::

    dbLoadRecords("table.db", "N=TST:,LBL1=Label A,LBL2=Label B,PO1=0,PO2=1")

.. literalinclude:: ../test/table.db
    :linenos:

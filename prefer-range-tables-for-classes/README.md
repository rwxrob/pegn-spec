# Prefer range tables for classes

Class implementations SHOULD use range tables for looking up values more efficiently and combining them with class maps that key to the name of the class. This prevents problems with languages that cannot use the lowercase class identifier otherwise. See Go `unicode` package for examples.

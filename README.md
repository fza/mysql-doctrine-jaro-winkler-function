# Jaro-Winkler distance function for Doctrine and MySQL

A tiny Doctrine extension for the [Jaro-Winkler distance](http://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance) algorithm to be used directly in DQL. The `JARO_WINKLER_SIMILARITY(s1, s2)` function returns a float between 0 and 1, where 0 means there's no similarity at all and 1 indicates an exact match.

Just for reference, there are plenty of alternative/additional algorithms to compute phonetic similarity. This is by all means not a complete list:

* [Damerau-Levenshtein distance](http://en.wikipedia.org/wiki/Damerau%E2%80%93Levenshtein_distance)
* [Levenshtein distance](http://en.wikipedia.org/wiki/Levenshtein_distance) (there is a [Doctrine extension](https://github.com/fza/mysql-doctrine-levenshtein-function) available as well) 
* [Soundex](http://en.wikipedia.org/wiki/Jaro%E2%80%93Winkler_distance)
* [Metaphone](http://en.wikipedia.org/wiki/Metaphone)

## Define MySQL function

* [Source](https://androidaddicted.wordpress.com/2010/06/01/jaro-winkler-sql-code/)

Execute the following commands to define the `JARO_WINKLER_SIMILARITY` function in the database. This needs to be done before you can use the function in any query.

```sql
DELIMITER $$

CREATE DEFINER =`root`@`localhost` FUNCTION `jaro_winkler_similarity`(
  in1 VARCHAR(255),
  in2 VARCHAR(255)
)
  RETURNS FLOAT
DETERMINISTIC
  BEGIN
#finestra:= search window, curString:= scanning cursor for the original string, curSub:= scanning cursor for the compared string
    DECLARE finestra, curString, curSub, maxSub, trasposizioni, prefixlen, maxPrefix INT;
    DECLARE char1, char2 CHAR(1);
    DECLARE common1, common2, old1, old2 VARCHAR(255);
    DECLARE trovato BOOLEAN;
    DECLARE returnValue, jaro FLOAT;
    SET maxPrefix = 6;
#from the original jaro - winkler algorithm
    SET common1 = "";
    SET common2 = "";
    SET finestra = (length(in1) + length(in2) - abs(length(in1) - length(in2))) DIV 4
                   + ((length(in1) + length(in2) - abs(length(in1) - length(in2))) / 2) MOD 2;
    SET old1 = in1;
    SET old2 = in2;

#calculating common letters vectors
    SET curString = 1;
    WHILE curString <= length(in1) AND (curString <= (length(in2) + finestra)) DO
      SET curSub = curstring - finestra;
      IF (curSub) < 1
      THEN
        SET curSub = 1;
      END IF;
      SET maxSub = curstring + finestra;
      IF (maxSub) > length(in2)
      THEN
        SET maxSub = length(in2);
      END IF;
      SET trovato = FALSE;
      WHILE curSub <= maxSub AND trovato = FALSE DO
        IF substr(in1, curString, 1) = substr(in2, curSub, 1)
        THEN
          SET common1 = concat(common1, substr(in1, curString, 1));
          SET in2 = concat(substr(in2, 1, curSub - 1), concat("0", substr(in2, curSub + 1, length(in2) - curSub + 1)));
          SET trovato = TRUE;
        END IF;
        SET curSub = curSub + 1;
      END WHILE;
      SET curString = curString + 1;
    END WHILE;
#back to the original string
    SET in2 = old2;
    SET curString = 1;
    WHILE curString <= length(in2) AND (curString <= (length(in1) + finestra)) DO
      SET curSub = curstring - finestra;
      IF (curSub) < 1
      THEN
        SET curSub = 1;
      END IF;
      SET maxSub = curstring + finestra;
      IF (maxSub) > length(in1)
      THEN
        SET maxSub = length(in1);
      END IF;
      SET trovato = FALSE;
      WHILE curSub <= maxSub AND trovato = FALSE DO
        IF substr(in2, curString, 1) = substr(in1, curSub, 1)
        THEN
          SET common2 = concat(common2, substr(in2, curString, 1));
          SET in1 = concat(substr(in1, 1, curSub - 1), concat("0", substr(in1, curSub + 1, length(in1) - curSub + 1)));
          SET trovato = TRUE;
        END IF;
        SET curSub = curSub + 1;
      END WHILE;
      SET curString = curString + 1;
    END WHILE;
#back to the original string
    SET in1 = old1;

#calculating jaro metric
    IF length(common1) <> length(common2)
    THEN SET jaro = 0;
    ELSEIF length(common1) = 0 OR length(common2) = 0
      THEN SET jaro = 0;
    ELSE
#calcolo la distanza di winkler
#passo 1: calcolo le trasposizioni
      SET trasposizioni = 0;
      SET curString = 1;
      WHILE curString <= length(common1) DO
        IF (substr(common1, curString, 1) <> substr(common2, curString, 1))
        THEN
          SET trasposizioni = trasposizioni + 1;
        END IF;
        SET curString = curString + 1;
      END WHILE;
      SET jaro =
      (
        length(common1) / length(in1) +
        length(common2) / length(in2) +
        (length(common1) - trasposizioni / 2) / length(common1)
      ) / 3;

    END IF;
#end if for jaro metric

#calculating common prefix for winkler metric
    SET prefixlen = 0;
    WHILE (substring(in1, prefixlen + 1, 1) = substring(in2, prefixlen + 1, 1)) AND (prefixlen < 6) DO
      SET prefixlen = prefixlen + 1;
    END WHILE;


#calculate jaro-winkler metric
    RETURN jaro + (prefixlen * 0.1 * (1 - jaro));
  END
```

## Symfony2 configuration

```yaml
# app/config/config.yml

doctrine:
  orm:
    entity_managers:
      default:
        dql:
          numeric_functions:
            jaro_winkler_similarity: Fza\MysqlDoctrineJaroWinklerSimilarityFunction\DQL\JaroWinklerSimilarityFunction
```

## Query example

```php
$em = $this->getEntityManager();
$query = $em->createQuery('SELECT u FROM User u WHERE JARO_WINKLER_SIMILARITY(u.name, :nameQuery) > :minSimilarity');
$query->setParameter('nameQuery', 'michael');
$query->setParameter('minSimilarity', 0.5)
$matchingUsers = $query->getResult();
```

## License

Copyright (c) 2015 [Felix Zandanel](http://felix.zandanel.me/)  
Licensed under the MIT license.

See LICENSE for more info.

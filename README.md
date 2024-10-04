# SQL-Interview-Questions

## Pattern Printing
**MySQL**:
*****
* *
* * *
* * * *
* * * * *

**code**:
```
DELIMETER $$
-- create procedure and take input value (R)
CREATE PROCEDURE PATTERN(IN R INT)
BEGIN
  DECLARE i INT DEFAULT 1;
  WHILE i<=R DO
    SELECT REPEAT('.',i);
    SET i=i+1;
  END WHILE;
END $$

DELIMITER ;

-- call procedure
CALL PATTERN(5);
```

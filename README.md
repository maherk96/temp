```java
egrep "OAG01VT04000T23|OAG01VT04000U3F|OAG01VT04000V3H|OAG01VT04001223|OAG01VT04001123|OAG01VT04001223|OAG01VT04001323|OAG01VT04001423|OAG01VT04001523" \
  fix_gateway.20251112_095400.log |
awk -F'|' '
{
  clord = ""
  msgtype = ""

  # find 11 and 35 fields in the line
  for (i = 1; i <= NF; i++) {
    if ($i ~ /^11=/) clord = substr($i, 4)
    if ($i ~ /^35=/) msgtype = substr($i, 4)
  }

  if (clord == "") next

  if (msgtype == "D") hasD[clord] = 1
  if (msgtype == "8") has8[clord] = 1
}
END {
  # print ClOrdIDs that have D but no 8
  for (id in hasD)
    if (!(id in has8))
      print id
}
' | sort

```

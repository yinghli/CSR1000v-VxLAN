Connect overlap subnet between Azure and on premise using VxLAN
==============================

Background<br>
--------------------
When customer migrate their application from on premise data center to Azure data center, it may have a short time that same IP subnet in both Azure and on premise side.<br>
By design, Azure didn’t support overlapping IP subnet. Below is a short term workaround by using VxLAN to bridge same IP subnet in both side.<br>

Topology<br>
-------------------


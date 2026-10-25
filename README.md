# Scampton the Great — modded data.win

`data.win.zip.part00..partNN` are a split zip of the modified `data.win`.

Restore (bash): `cat data.win.zip.part* > data.win.zip && unzip data.win.zip`
Restore (PowerShell): `cmd /c copy /b data.win.zip.part00+data.win.zip.part01+data.win.zip.part02+data.win.zip.part03+data.win.zip.part04+data.win.zip.part05 data.win.zip; Expand-Archive data.win.zip .`

Replace `data.win` in the original game folder with the extracted file.

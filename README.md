# AA2 – CUPS: Servidor d'impressió amb impressora virtual (cups-pdf)

**Autor:** Guillermo
**Assignatura:** Administració de sistemes
**Activitat:** AA2 – Prova de concepte (PoC) de CUPS

## Descripció

Aquesta PoC mostra com instal·lar i configurar un servidor d'impressió **CUPS** en un Ubuntu Server i com compartir-ne una impressora virtual (`cups-pdf`) amb un client **Zorin OS**. La impressora virtual no imprimeix en paper: genera un fitxer PDF al servidor, cosa que permet simular una impressora de xarxa sense maquinari físic.

## Escenari

| Màquina | Sistema | Xarxa | Rol |
|---|---|---|---|
| SERVER | Ubuntu Server | NAT + Host-Only (`192.168.10.1`) | Servidor d'impressió CUPS |
| Client | Zorin OS Desktop | Host-Only | Client d'impressió |

És el mateix escenari que es va utilitzar a la PoC de NFS.

## Contingut del repositori

```
.
├── README.md        # Aquest fitxer (resum i índex)
├── AA2-CUPS.md      # Documentació completa de la pràctica
└── images/          # Captures de pantalla
```

## Índex de la documentació (AA2-CUPS.md)

1. Instal·lació de CUPS
2. Instal·lació de la impressora virtual cups-pdf
3. Configuració de l'administració de CUPS
4. Compartició de la impressora
5. Configuració del client Zorin OS
6. Proves d'impressió
7. Comprovació dels documents generats
8. Conclusions

## Comandes principals

```bash
sudo apt update
sudo apt install cups
sudo apt install printer-driver-cups-pdf
sudo nano /etc/cups/cupsd.conf
sudo systemctl restart cups
sudo usermod -aG lpadmin vboxuser
sudo lpadmin -p PDF_Printer -v cups-pdf:/ -m raw -E
sudo cupsenable PDF_Printer
sudo cupsaccept PDF_Printer
```

Per als detalls i les captures, vegeu [AA2-CUPS.md](AA2-CUPS.md).

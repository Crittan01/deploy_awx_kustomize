Certificados Necesarios para AWX

Descripción General
Este documento describe los certificados requeridos para la correcta configuración de AWX con conexión segura (HTTPS y LDAPS). Se detallan los archivos involucrados, sus propósitos y la forma en que deben ser utilizados.

Estructura de Certificados
Los siguientes archivos son necesarios para la implementación segura de AWX:

Archivo Descripción
awx.empresa.cer: Certificado SSL del servidor AWX, emitido por la CA interna (CAinterna.cer).
awx.empresa.key: Clave privada correspondiente al certificado de AWX. Debe mantenerse segura.

CAinterna.cer: Certificado de la Autoridad Certificadora Interna (CA Interna) que firmó el certificado de AWX.

awx_full.crt: Certificado completo en formato PEM, incluye la cadena de confianza completa (AWX + CA Interna).

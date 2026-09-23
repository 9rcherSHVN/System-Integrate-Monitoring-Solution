# Certificate Expiry Response Runbook

## Trigger Conditions

- Certificate expired.
- Certificate expires within 7, 14, or 30 days.
- IIS-bound certificate missing a private key.
- Certificate collector has failed or has not reported.

## Initial Triage

1. Open the **ERP Certificate Monitoring** Workbook.
2. Identify:
   - Server name.
   - Certificate subject and SAN.
   - Thumbprint.
   - Expiry date.
   - IIS binding, if present.
   - Certificate owner/application owner.
3. Confirm whether the certificate is actively bound to an ERP or middleware endpoint.
4. Check whether a replacement certificate already exists in `Cert:\LocalMachine\WebHosting`.
5. Confirm certificate chain, private-key association, and validity dates.

## Resolution

1. Obtain or issue replacement certificate through the approved PKI/renewal process.
2. Import into the correct Local Machine certificate store.
3. Confirm private key is present.
4. Apply or update the IIS HTTPS binding.
5. Restart/reload application or IIS only if required by the application/vendor procedure.
6. Test the endpoint from an approved monitoring/client network.
7. Run the certificate collector manually.
8. Confirm the old alert resolves and the workbook shows the replacement certificate.

## Evidence Required

- Replacement certificate thumbprint.
- New certificate expiry date.
- Binding validation result.
- API/HTTPS validation result.
- Change record reference.

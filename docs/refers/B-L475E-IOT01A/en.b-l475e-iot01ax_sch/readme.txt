******************************************************************************
* @file    readme.txt
* @author  MCD Application Team
* @version V2.0
* @date    21-June-2017  
* @brief   b-l475e-iot01ax Schematics files package
******************************************************************************
* COPYRIGHT(c) 2017 STMicroelectronics
*
* The Open Platform License Agreement (“Agreement”) is a binding legal contract
* between you ("You") and STMicroelectronics International N.V. (“ST”), a
* company incorporated under the laws of the Netherlands acting for the purpose
* of this Agreement through its Swiss branch 39, Chemin du Champ des Filles,
* 1228 Plan-les-Ouates, Geneva, Switzerland.
*
* By using the enclosed reference designs, schematics, PC board layouts, and
* documentation, in hardcopy or CAD tool file format (collectively, the
* “Reference Material”), You are agreeing to be bound by the terms and
* conditions of this Agreement. Do not use the Reference Material until You
* have read and agreed to this Agreement terms and conditions. The use of
* the Reference Material automatically implies the acceptance of the Agreement
* terms and conditions.
*
* The complete Open Platform License Agreement can be found on www.st.com/opla.
******************************************************************************
*
******************************************************************************
* History
* V1.0	30-March-2017		MB1297C BOM (initial BOM)
* V2.0	21-June-2017		MB1297D BOM    
******************************************************************************
* 
* Update for revision D:
*
* - No BOM changes compared to the MB1297 C-01 BOM, 
* ie C53 = 10pF, C71 = 10pF and STSAFE-A100 (U9 component) not fitted.
* 
* - Two pcb changes compared to the MB1297 C-01 pcb:
* 	° The reset connexion between STM32L4 and the ST-LINK MCU (STM32F103) 
*	is implemented of the MB1297 rev D
*	° The pcb below the Wi-Fi antenna has been removed to have more Wi-Fi 
*	radiated output power
* 
*
* Important note: Note: Firmware revision inside the Wi-Fi module must be: C3.5.2.3.BETA9.
* The Wi-Fi module maximum output power is then limited to 9 dBm to fulfill FCC/IC/CE requirements
*
******************************************************************************  

******************* (C) COPYRIGHT 2017 STMicroelectronics *****END OF FILE

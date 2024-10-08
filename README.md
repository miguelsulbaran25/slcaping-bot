//+------------------------------------------------------------------+
//|                                                      TIME EA.mq5 |
//|                                  Copyright 2024, MetaQuotes Ltd. |
//|                                             https://www.mql5.com |
//+------------------------------------------------------------------+
#property copyright "Copyright 2024, MetaQuotes Ltd."
#property link      "https://www.mql5.com"
#property version   "1.02"

#include <Trade/Trade.mqh>
#include <ChartObjects\ChartObjectsTxtControls.mqh>
//+------------------------------------------------------------------+
//| Inputs                                                           |
//+------------------------------------------------------------------+
input group "==== Entradas Generales ===="
input int InpMagicNumber = 123456;                    // Magic Number para identificar las órdenes del EA
input double InpPercent = 3;                          // Porcentaje de riesgo en relación al balance de la cuenta
input int MaxSpread = 2;                              // Límite máximo de spread en puntos
input int InpTp = 200;                                // Puntos de Take Profit
input int InpSl = 200;                                // Puntos de Stop Loss
input ENUM_TIMEFRAMES TimeFrame = PERIOD_CURRENT;     // Marco de tiempo a utilizar
input group "==== Trailing Stop ===="
input bool EnableTrailingStop = true;                 // Activar/Desactivar el Trailing Stop
input int InpTrailingStop = 15;                       // Puntos del Trailing Stop inicial
input int TslPoints = 10;                             // Puntos del Trailing Stop por debajo/encima del precio
input group "==== Cierre de Operaciones ===="
input bool InpCloseAtEnd = false;                     // Cerrar todas las posiciones a final de tiempo
input group "==== Periodo de Tiempo ===="
input int InpBarsToAnalyze = 200;                     // Cantidad de velas a analizar
input int ExpirationsBars = 100;                      // Número de barras para expiración de órdenes
input int OrderDiskPoints = 100;                      // Puntos de separación para órdenes
input int SHInput = 0;                                // Hora de Inicio de operación (0-23)
input int EHInput = 0;                                // Hora de Fin de operación (0-23)
input group "==== Personalización ===="
input string TradeComment = "Comentario del EA";      // Comentario en las órdenes del EA

// Variables generales
CTrade trade; // Objeto para operaciones de trading
CPositionInfo pos; // Objeto para información de posiciones
COrderInfo ord; // Objeto para información de órdenes
int BarsN = 5; // Número de barras para análisis
// Variable global para almacenar el balance inicial
double balance_inicial;

//+------------------------------------------------------------------+
//| Expert initialization function                                   |
//+------------------------------------------------------------------+
int OnInit()
{
   // Obtener el balance inicial de la cuenta
   balance_inicial = AccountInfoDouble(ACCOUNT_BALANCE);
   
   // Crear el panel inicial
   CrearPanel();

   /*// Verificación para detectar si el gráfico está en tiempo real
    if (MQLInfoInteger(MQL_TESTER) == false)
    {
        // Si no está en modo prueba, detener el bot y mostrar un mensaje
        Print("Este bot solo se puede utilizar en el modo de prueba. Deteniendo el EA.");
        return(INIT_FAILED); // Finaliza la ejecución del EA
    }*/
    
   trade.SetExpertMagicNumber(InpMagicNumber); // Configura el Magic Number para las operaciones
   ChartSetInteger(0, CHART_SHOW_GRID, false); // Desactiva la visualización de la cuadrícula en el gráfico

   return(INIT_SUCCEEDED); // Indica que la inicialización fue exitosa
}

//+------------------------------------------------------------------+
//| Expert deinitialization function                                 |
//+------------------------------------------------------------------+
void OnDeinit(const int reason)
{
   // No se realiza ninguna acción al desinicializar el EA
}

//+------------------------------------------------------------------+
//| Expert tick function                                             |
//+------------------------------------------------------------------+
void OnTick()
{
     ActualizarPanelBalance();
   // Llama a la función para gestionar el trailing stop
   TrailinStop();
   // Verifica si hay una nueva barra en el gráfico
   if (!IsNewBar()) return;

   MqlDateTime time;
   TimeToStruct(TimeCurrent(), time); // Convierte el tiempo actual a estructura MqlDateTime

   int Hournow = time.hour; // Hora actual

   // Controla el horario de operación considerando el cruce de medianoche
   bool isWithinOperatingHours = false;
   if (SHInput < EHInput) {
        // Caso normal, sin cruce de medianoche
        isWithinOperatingHours = (Hournow >= SHInput && Hournow < EHInput);
    } else if (SHInput > EHInput) {
        // Caso con cruce de medianoche
        isWithinOperatingHours = (Hournow >= SHInput || Hournow < EHInput);
    } else {
        // Caso especial donde SHInput == EHInput, significando 24 horas de operación
        isWithinOperatingHours = true;
    }

    // Si el tiempo actual está fuera del rango operativo
    if (!isWithinOperatingHours) {
        if (InpCloseAtEnd) {
            CloseAllPositions(); // Cierra todas las posiciones si está habilitado
        }
        return; // Sale de la función sin hacer más nada
    }

    int BuyTotal = 0;
    int SellTotal = 0;

    // Verifica las posiciones abiertas
    for (int i = PositionsTotal() - 1; i >= 0; i--)
    {
        pos.SelectByIndex(i);
        if (pos.PositionType() == POSITION_TYPE_BUY && pos.Symbol() == _Symbol && pos.Magic() == InpMagicNumber) BuyTotal++;
        if (pos.PositionType() == POSITION_TYPE_SELL && pos.Symbol() == _Symbol && pos.Magic() == InpMagicNumber) SellTotal++;
    }

    // Verifica las órdenes pendientes
    for (int i = OrdersTotal() - 1; i >= 0; i--)
    {
        ord.SelectByIndex(i);
        if (ord.OrderType() == ORDER_TYPE_BUY_STOP && ord.Symbol() == _Symbol && ord.Magic() == InpMagicNumber) BuyTotal++;
        if (ord.OrderType() == ORDER_TYPE_SELL_STOP && ord.Symbol() == _Symbol && ord.Magic() == InpMagicNumber) SellTotal++;
    }

    // Si no hay órdenes de compra abiertas, envía una orden de compra
    if (BuyTotal <= 0)
    {
        double high = findHigh(); // Encuentra el valor alto para la orden de compra
        if (high > 0)
        {
            SendBuyOrder(high); // Envía la orden de compra
        }
    }

    // Si no hay órdenes de venta abiertas, envía una orden de venta
    if (SellTotal <= 0)
    {
        double low = findLow(); // Encuentra el valor bajo para la orden de venta
        if (low > 0)
        {
            SendSellOrder(low); // Envía la orden de venta
        }
    }
}

//+------------------------------------------------------------------+
//| Funciones auxiliares                                              |
//+------------------------------------------------------------------+

// Encuentra el máximo valor alto de las últimas barras
double findHigh()
{
   double highestHigh = 0;
   for (int i = 0; i < InpBarsToAnalyze; i++)
   {
      double high = iHigh(_Symbol, TimeFrame, i);
      if (i > BarsN && iHighest(_Symbol, TimeFrame, MODE_HIGH, BarsN * 2 + 1, i - BarsN) == i)
      {
         if (high > highestHigh)
         {
            return high;
         }
      }
      highestHigh = MathMax(high, highestHigh);
   }
   return -1;
}

// Encuentra el valor más bajo de las últimas barras
double findLow()
{
   double lowestLow = DBL_MAX;
   for (int i = 0; i < InpBarsToAnalyze; i++)
   {
      double low = iLow(_Symbol, TimeFrame, i);
      if (i > BarsN && iLowest(_Symbol, TimeFrame, MODE_LOW, BarsN * 2 + 1, i - BarsN) == i)
      {
         if (low < lowestLow)
         {
            return low;
         }
      }
      lowestLow = MathMin(low, lowestLow);
   }
   return -1;
}

// Verifica si hay una nueva barra
bool IsNewBar()
{
   static datetime previusTime = 0;
   datetime curretTime = iTime(_Symbol, TimeFrame, 0);
   if (previusTime != curretTime)
   {
      previusTime = curretTime;
      return true;
   }
   return false;
}

// Enviar orden de compra
void SendBuyOrder(double entry)
{
    // Obtener el spread en puntos como long
    long spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
    
    // Convertir MaxSpread a long para la comparación
    if (spread > (long)MaxSpread) return; // Verificar si el spread es mayor que el límite permitido

    double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
    if (ask > entry - OrderDiskPoints * _Point) return;

    double tp = entry + InpTp * _Point; // Puntos de Take Profit
    double sl = entry - InpSl * _Point; // Puntos de Stop Loss

    double lots = 0.01; // Tamaño inicial del lote
    if (InpPercent > 0) lots = calcLots(entry - sl); // Calcular el tamaño del lote basado en el riesgo

    datetime expiration = iTime(_Symbol, TimeFrame, 0) + ExpirationsBars * PeriodSeconds(TimeFrame); // Tiempo de expiración de la orden

    trade.BuyStop(lots, entry, _Symbol, sl, tp, ORDER_TIME_SPECIFIED, expiration); // Envía la orden de compra
}

// Enviar orden de venta
void SendSellOrder(double entry)
{
    // Obtener el spread en puntos como long
    long spread = SymbolInfoInteger(_Symbol, SYMBOL_SPREAD);
    
    // Convertir MaxSpread a long para la comparación
    if (spread > (long)MaxSpread) return; // Verificar si el spread es mayor que el límite permitido

    double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);
    if (bid < entry - OrderDiskPoints * _Point) return;

    double tp = entry - InpTp * _Point; // Puntos de Take Profit
    double sl = entry + InpSl * _Point; // Puntos de Stop Loss

    double lots = 0.01; // Tamaño inicial del lote
    if (InpPercent > 0) lots = calcLots(sl - entry); // Calcular el tamaño del lote basado en el riesgo

    datetime expiration = iTime(_Symbol, TimeFrame, 0) + ExpirationsBars * PeriodSeconds(TimeFrame); // Tiempo de expiración de la orden

    trade.SellStop(lots, entry, _Symbol, sl, tp, ORDER_TIME_SPECIFIED, expiration); // Envía la orden de venta
}

// Cálculo del tamaño de los lotes basado en el riesgo
double calcLots(double slPoints)
{
   double risk = AccountInfoDouble(ACCOUNT_BALANCE) * InpPercent / 100; // Riesgo en base al porcentaje

   double ticksize = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_SIZE);
   double tickvalue = SymbolInfoDouble(_Symbol, SYMBOL_TRADE_TICK_VALUE);
   double lotstep = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_STEP);
   double minvolume = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN);
   double maxvolume = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX);
   double volumelimit = SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_LIMIT);

   double moneyPerLotstep = slPoints / ticksize * tickvalue * lotstep; // Valor en dinero de un paso de lote
   double lots = MathFloor(risk / moneyPerLotstep) * lotstep; // Cálculo de lotes

   // Limitar el tamaño del lote según los límites del símbolo
   if (volumelimit != 0) lots = MathMin(lots, volumelimit);
   if (maxvolume != 0) lots = MathMin(lots, SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MAX));
   if (minvolume != 0) lots = MathMax(lots, SymbolInfoDouble(_Symbol, SYMBOL_VOLUME_MIN));
   lots = NormalizeDouble(lots, 2); // Normaliza el tamaño del lote

   return lots;
}

// Cerrar todas las posiciones abiertas
void CloseAllPositions()
{
    for (int i = PositionsTotal() - 1; i >= 0; i--)
    {
        if (pos.SelectByIndex(i) && pos.Symbol() == _Symbol && pos.Magic() == InpMagicNumber)
        {
            ulong ticket = pos.Ticket();
            if (pos.PositionType() == POSITION_TYPE_BUY)
            {
                trade.PositionClose(ticket); // Cierra posición de compra
            }
            else if (pos.PositionType() == POSITION_TYPE_SELL)
            {
                trade.PositionClose(ticket); // Cierra posición de venta
            }
        }
    }
}

// Gestión del trailing stop
void TrailinStop()
{
    if (!EnableTrailingStop) return; // Si el trailing stop está desactivado, salir de la función

    double sl = 0;
    double tp = 0;

    double ask = SymbolInfoDouble(_Symbol, SYMBOL_ASK);
    double bid = SymbolInfoDouble(_Symbol, SYMBOL_BID);

    if (TslPoints <= 0) return; // Verificación adicional para TslPoints

    for (int i = PositionsTotal() - 1; i >= 0; i--)
    {
        if (pos.SelectByIndex(i))
        {
            ulong ticket = pos.Ticket();
            if (pos.Magic() == InpMagicNumber && pos.Symbol() == _Symbol)
            {
                if (pos.PositionType() == POSITION_TYPE_BUY)
                {
                    if (bid - pos.PriceOpen() > InpTrailingStop * _Point)
                    {
                        tp = pos.TakeProfit();
                        sl = bid - (TslPoints * _Point);
                        if (sl > pos.StopLoss())
                        {
                            trade.PositionModify(ticket, sl, tp); // Modifica el stop loss y take profit
                        }
                    }
                }
                if (pos.PositionType() == POSITION_TYPE_SELL)
                {
                    if (pos.PriceOpen() - ask > InpTrailingStop * _Point)
                    {
                        tp = pos.TakeProfit();
                        sl = ask + (TslPoints * _Point);
                        if (sl < pos.StopLoss())
                        {
                            trade.PositionModify(ticket, sl, tp); // Modifica el stop loss y take profit
                        }
                    }
                }
            }
        }
    }
}

// Función para crear el panel
void CrearPanel()
{
   // Verificamos si ya existe el objeto, si no lo creamos
   if (ObjectFind(0, "PanelBalance") == -1)
   {
      // Crear un objeto rectángulo como fondo del panel
      ObjectCreate(0, "PanelBalance", OBJ_RECTANGLE_LABEL, 0, 0, 0);
      ObjectSetInteger(0, "PanelBalance", OBJPROP_CORNER, CORNER_LEFT_UPPER); // Esquina superior izquierda
      ObjectSetInteger(0, "PanelBalance", OBJPROP_XDISTANCE, 10); // Distancia del borde izquierdo
      ObjectSetInteger(0, "PanelBalance", OBJPROP_YDISTANCE, 10); // Distancia del borde superior
      ObjectSetInteger(0, "PanelBalance", OBJPROP_COLOR, clrDarkGray); // Color del fondo
      ObjectSetInteger(0, "PanelBalance", OBJPROP_STYLE, STYLE_SOLID); // Estilo sólido
      ObjectSetInteger(0, "PanelBalance", OBJPROP_SELECTABLE, false); // No seleccionable
      ObjectSetInteger(0, "PanelBalance", OBJPROP_HIDDEN, true); // Escondido en la lista de objetos
      ObjectSetInteger(0, "PanelBalance", OBJPROP_BACK, true); // En el fondo

      // Ajustar el ancho del panel
      ObjectSetInteger(0, "PanelBalance", OBJPROP_WIDTH, 150);  
      ObjectSetDouble(0, "PanelBalance", OBJPROP_SCALE, 1.0); // Escala para ajuste del tamaño
   }

   // Verificamos si ya existe el texto del balance, si no lo creamos
   if (ObjectFind(0, "TextoBalance") == -1)
   {
      ObjectCreate(0, "TextoBalance", OBJ_LABEL, 0, 0, 0);
      ObjectSetInteger(0, "TextoBalance", OBJPROP_CORNER, CORNER_LEFT_UPPER); // Esquina superior izquierda
      ObjectSetInteger(0, "TextoBalance", OBJPROP_XDISTANCE, 15); // Distancia del borde izquierdo
      ObjectSetInteger(0, "TextoBalance", OBJPROP_YDISTANCE, 20); // Distancia del borde superior (dentro del panel)
      ObjectSetInteger(0, "TextoBalance", OBJPROP_FONTSIZE, 14); // Tamaño de la fuente
      ObjectSetInteger(0, "TextoBalance", OBJPROP_BACK, true); // En el fondo
   }
}

// Función para actualizar el panel con las ganancias o pérdidas acumuladas
void ActualizarPanelBalance()
{
   // Obtener el equity actual de la cuenta
   double equity = AccountInfoDouble(ACCOUNT_EQUITY);
   
   // Calcular la ganancia o pérdida acumulada
   double ganancia_perdida = equity - balance_inicial;
   
   // Definir el texto que se mostrará
   string textoBalance;
   
   // Si hay ganancia, mostrar en verde con el valor positivo
   if (ganancia_perdida > 0)
   {
      textoBalance = StringFormat("Ender lo mama bien: %.2f USD", ganancia_perdida);
      ObjectSetInteger(0, "TextoBalance", OBJPROP_COLOR, clrGreen); // Color verde para ganancia
   }
   // Si hay pérdida, mostrar en rojo con el valor negativo
   else if (ganancia_perdida < 0)
   {
      textoBalance = StringFormat("Ender lo mama mal: %.2f USD", MathAbs(ganancia_perdida)); // MathAbs para mostrar el valor absoluto
      ObjectSetInteger(0, "TextoBalance", OBJPROP_COLOR, clrRed); // Color rojo para pérdida
   }
   // Si no hay ni ganancia ni pérdida, mostrar en blanco
   else
   {
      textoBalance = "Sin cambios";
      ObjectSetInteger(0, "TextoBalance", OBJPROP_COLOR, clrWhite); // Color blanco para sin cambios
   }

   // Actualizar el texto con la ganancia o pérdida acumulada
   ObjectSetString(0, "TextoBalance", OBJPROP_TEXT, textoBalance);
}

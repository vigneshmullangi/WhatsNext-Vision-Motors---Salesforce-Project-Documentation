## **VehicleOrderTriggerHandler:-**

##  



public class VehicleOrderTriggerHandler {

&nbsp;

&nbsp;   public static void handleTrigger(List<Vehicle\_Order\_\_c> newOrders, Map<Id, Vehicle\_Order\_\_c> oldOrders, Boolean isBefore, Boolean isAfter, Boolean isInsert, Boolean isUpdate) {

&nbsp;       if (isBefore \&\& (isInsert || isUpdate)) {

&nbsp;           preventOrderIfOutOfStock(newOrders);

&nbsp;       }

&nbsp;

&nbsp;       if (isAfter \&\& (isInsert || isUpdate)) {

&nbsp;           updateStockOnOrderPlacement(newOrders);

&nbsp;       }

&nbsp;   }

&nbsp;

&nbsp;   // ❌ Prevent placing an order if stock is zero

&nbsp;   private static void preventOrderIfOutOfStock(List<Vehicle\_Order\_\_c> orders) {

&nbsp;       Set<Id> vehicleIds = new Set<Id>();

&nbsp;       for (Vehicle\_Order\_\_c order : orders) {

&nbsp;           if (order.Vehicle\_\_c != null) {

&nbsp;               vehicleIds.add(order.Vehicle\_\_c);

&nbsp;           }

&nbsp;       }

&nbsp;

&nbsp;       if (!vehicleIds.isEmpty()) {

&nbsp;           Map<Id, Vehicle\_\_c> vehicleStockMap = new Map<Id, Vehicle\_\_c>(

&nbsp;               \[SELECT Id, Stock\_Quantity\_\_c FROM Vehicle\_\_c WHERE Id IN :vehicleIds]

&nbsp;           );

&nbsp;

&nbsp;           for (Vehicle\_Order\_\_c order : orders) {

&nbsp;               Vehicle\_\_c vehicle = vehicleStockMap.get(order.Vehicle\_\_c);

&nbsp;               if (vehicle != null \&\& vehicle.Stock\_Quantity\_\_c <= 0) {

&nbsp;                   order.addError('This vehicle is out of stock. Order cannot be placed.');

&nbsp;               }

&nbsp;           }

&nbsp;       }

&nbsp;   }

&nbsp;

&nbsp;   // ✅ Decrease stock when an order is confirmed

&nbsp;   private static void updateStockOnOrderPlacement(List<Vehicle\_Order\_\_c> orders) {

&nbsp;       Set<Id> vehicleIds = new Set<Id>();

&nbsp;       for (Vehicle\_Order\_\_c order : orders) {

&nbsp;           if (order.Vehicle\_\_c != null \&\& order.Status\_\_c == 'Confirmed') {

&nbsp;               vehicleIds.add(order.Vehicle\_\_c);

&nbsp;           }

&nbsp;       }

&nbsp;

&nbsp;       if (!vehicleIds.isEmpty()) {

&nbsp;           Map<Id, Vehicle\_\_c> vehicleStockMap = new Map<Id, Vehicle\_\_c>(

&nbsp;               \[SELECT Id, Stock\_Quantity\_\_c FROM Vehicle\_\_c WHERE Id IN :vehicleIds]

&nbsp;           );

&nbsp;

&nbsp;           List<Vehicle\_\_c> vehiclesToUpdate = new List<Vehicle\_\_c>();

&nbsp;           for (Vehicle\_Order\_\_c order : orders) {

&nbsp;               Vehicle\_\_c vehicle = vehicleStockMap.get(order.Vehicle\_\_c);

&nbsp;               if (vehicle != null \&\& vehicle.Stock\_Quantity\_\_c > 0) {

&nbsp;                   vehicle.Stock\_Quantity\_\_c -= 1;

&nbsp;                   vehiclesToUpdate.add(vehicle);

&nbsp;               }

&nbsp;           }

&nbsp;

&nbsp;           if (!vehiclesToUpdate.isEmpty()) {

&nbsp;               update vehiclesToUpdate;

&nbsp;           }

&nbsp;       }

&nbsp;   }

}



## **VehicleOrderTrigger:-**



trigger VehicleOrderTrigger on Vehicle\_Order\_\_c (before insert, before update, after insert, after update) {

&nbsp;   VehicleOrderTriggerHandler.handleTrigger(Trigger.new, Trigger.oldMap, Trigger.isBefore, Trigger.isAfter, Trigger.isInsert, Trigger.isUpdate);

}





## **VehicleOrderBatch:-**



&nbsp;



global class VehicleOrderBatch implements Database.Batchable<sObject> {

&nbsp;

&nbsp;   global Database.QueryLocator start(Database.BatchableContext bc) {

&nbsp;       return Database.getQueryLocator(\[

&nbsp;           SELECT Id, Status\_\_c, Vehicle\_\_c FROM Vehicle\_Order\_\_c WHERE Status\_\_c = 'Pending'

&nbsp;       ]);

&nbsp;   }

&nbsp;

&nbsp;   global void execute(Database.BatchableContext bc, List<Vehicle\_Order\_\_c> orderList) {

&nbsp;       Set<Id> vehicleIds = new Set<Id>();

&nbsp;       for (Vehicle\_Order\_\_c order : orderList) {

&nbsp;           if (order.Vehicle\_\_c != null) {

&nbsp;               vehicleIds.add(order.Vehicle\_\_c);

&nbsp;           }

&nbsp;       }

&nbsp;

&nbsp;       if (!vehicleIds.isEmpty()) {

&nbsp;           Map<Id, Vehicle\_\_c> vehicleStockMap = new Map<Id, Vehicle\_\_c>(

&nbsp;               \[SELECT Id, Stock\_Quantity\_\_c FROM Vehicle\_\_c WHERE Id IN :vehicleIds]

&nbsp;           );

&nbsp;

&nbsp;           List<Vehicle\_Order\_\_c> ordersToUpdate = new List<Vehicle\_Order\_\_c>();

&nbsp;           List<Vehicle\_\_c> vehiclesToUpdate = new List<Vehicle\_\_c>();

&nbsp;

&nbsp;           for (Vehicle\_Order\_\_c order : orderList) {

&nbsp;               Vehicle\_\_c vehicle = vehicleStockMap.get(order.Vehicle\_\_c);

&nbsp;               if (vehicle != null \&\& vehicle.Stock\_Quantity\_\_c > 0) {

&nbsp;                   order.Status\_\_c = 'Confirmed';

&nbsp;                   vehicle.Stock\_Quantity\_\_c -= 1;

&nbsp;                   ordersToUpdate.add(order);

&nbsp;                   vehiclesToUpdate.add(vehicle);

&nbsp;               }

&nbsp;           }

&nbsp;

&nbsp;           if (!ordersToUpdate.isEmpty()) update ordersToUpdate;

&nbsp;           if (!vehiclesToUpdate.isEmpty()) update vehiclesToUpdate;

&nbsp;       }

&nbsp;   }

&nbsp;

&nbsp;   global void finish(Database.BatchableContext bc) {

&nbsp;       System.debug('Vehicle order batch job completed.');

&nbsp;   }

}

&nbsp;

## 

## **VehicleOrderBatchScheduler:-**



&nbsp;



global class VehicleOrderBatchScheduler implements Schedulable {

&nbsp;   global void execute(SchedulableContext sc) {

&nbsp;       VehicleOrderBatch batchJob = new VehicleOrderBatch();

&nbsp;       Database.executeBatch(batchJob, 50); // 50 = batch size

&nbsp;   }

}


```Swift
//
//  BaseService.swift
//  KliquedUp-iOS
//
//  Created by James Hays on 2/9/16.
//  Copyright © 2016 Klique Inc. All rights reserved.
//

import Foundation
import Alamofire
import ObjectMapper
import RealmSwift
import AlamofireObjectMapper
import AlamofireImage
import OMGHTTPURLRQ
import Crashlytics

class KliqueService {
    
    enum ServiceEnvironment: String {
        case Development
        case Staging
        case Production
    }
    
    
    enum JSONConvertionType {
        
        case jsonForUpdate
        case jsonForCreate
    }
    
    
    static var credential: OAuthCredential? = nil
    static let dateFormatter: String = "yyyy-MM-dd'T'HH:mm:ss.SSS'Z'"
    static let slowDownloadThreshold: TimeInterval = 1.5 // seconds
    
    class func setCredential(credential: OAuthCredential?) {
        
        KliqueService.credential = credential
        NotificationCenter.default.post(name: kCredentialRefreshedAndAuthenticatedUser, object:nil)
    }
    
    var serviceEnvironment: KliqueService.ServiceEnvironment = .Production
    
    init() {
        let svcEnv = KliqueSettings.sharedInstance.serviceEnvironment
        switch svcEnv {
        case KliqueSettings.ServiceEnvironment.development:
            self.serviceEnvironment = .Development
            break
        case KliqueSettings.ServiceEnvironment.staging:
            self.serviceEnvironment = .Staging
            break
        case KliqueSettings.ServiceEnvironment.production:
            fallthrough
        default:
            self.serviceEnvironment = .Production
        }
        //print("KliqueService using base url \(self.serviceEnvironment.rawValue)")
    }
    
    func servicePath(endpoint: String) -> String {
        
        let baseUrl : String!
        switch serviceEnvironment {
            
        case .Development:
            baseUrl = "https://devapi.kliquedup.org:8080"
        case .Staging:
            baseUrl = "https://stageapi.kliquedup.org:8080"
        case .Production:
            baseUrl  = "https://api.kliquedup.org:8080"
        }
        
        return baseUrl + endpoint;
    }
    
    func getEntitiesAtPath<T:KliqueObject>(persist: Bool, urlString: String, path: String,
                           parameters: Parameters?,
                           completion:@escaping ([T],Error?) -> ()) -> Alamofire.Request {
        
        let endpoint: String = self.servicePath(endpoint: urlString)
        let request = self.createRequest(method: .get, URLString: endpoint, parameters: parameters)
            .responseArray(keyPath: path) { (dataResponse: DataResponse<[T]>) in
                self.processArrayResponse(persist: true, dataResponse: dataResponse,
                                          completion: { (entities, error) -> () in
                                            completion(entities , error)
                })
        }
        return request
    }
    
    func getEntities<T:KliqueObject>(urlString: String, parameters: Parameters?,
                     completion:@escaping ([T],Error?) -> ()) {
        getEntities(persist: true, urlString: urlString, parameters: parameters, completion: completion)
    }
    
    func getEntities<T:KliqueObject>(persist: Bool, urlString: String, parameters: Parameters?,
                     completion:@escaping ([T],Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        self.createRequest(method: .get, URLString: endpoint, parameters: parameters)
            .responseArray { (dataResponse: DataResponse<[T]>) in
                self.processArrayResponse(persist: true, dataResponse: dataResponse,
                                          completion: { (entities, error) -> () in
                                            completion(entities , error)
                })
        }
    }
    
    func postEntity<T:KliqueObject>(urlString: String, object: KliqueObject?,
                    completion:@escaping (T?,Error?) -> ()) {
        postEntity(persist: true, urlString: urlString, object: object,
                   completion: completion)
    }
    
    func postEntity<T:KliqueObject>(persist: Bool, urlString: String, object: KliqueObject?,
                    jsonCovertion:JSONConvertionType,
                    completion:@escaping (T?,Error?) -> ()) {
        
        let endpoint: String = self.servicePath(endpoint: urlString)
        let objectJSON : [String:Any]?
        if jsonCovertion == .jsonForUpdate {
            
            objectJSON =  object?.toJSONForUpdate()
        }
        else {
            
            objectJSON =  object?.toJSONForCreate()
        }
        //        debugPrint(objectJSON)
        
        self.createRequest(method: .post, URLString: endpoint, parameters: objectJSON)
            .responseObject { (dataResponse: DataResponse<T>) in
                
                self.processObjectResponse(persist: persist, dataResponse: dataResponse,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func postEntity<T:KliqueObject>(persist: Bool, urlString: String, object: KliqueObject?,
                    completion:@escaping (T?,Error?) -> ()) {
        self.postEntity(persist: persist, urlString: urlString, object: object,
                        jsonCovertion: JSONConvertionType.jsonForUpdate, completion: completion)
    }
    
    func postEntity<T:KliqueObject>(urlString: String, parameters:[String : Any]?,
                    completion:@escaping (T?,Error?) -> ()) {
        postEntity(persist: true, urlString: urlString, parameters: parameters, completion: completion)
    }
    
    
    func postEntity<T:KliqueObject>(persist: Bool, urlString: String, parameters:Parameters?,
                    completion:@escaping (T?,Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        
        self.createRequest(method: .post, URLString: endpoint, parameters: parameters, headers: [:],
                           encoding: JSONEncoding.default)
            .responseObject { (dataResponse: DataResponse<T>) in
                if let body = dataResponse.request?.httpBody {
                    debugPrint("POST params: \(String(describing: String(data: body, encoding: .utf8)))")
                }
                self.processObjectResponse(persist: persist, dataResponse: dataResponse,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func postEntities<T:KliqueObject>(urlString: String, object: KliqueObject?,
                      completion:@escaping ([T],Error?) -> ()) {
        postEntities(persist: true, urlString: urlString, object: object, completion: completion)
    }
    
    func postEntities<T:KliqueObject>(persist: Bool, urlString: String, object: KliqueObject?,
                      completion:@escaping ([T],Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        let objectJSON =  object?.toJSONForUpdate()
        self.createRequest(method: .post, URLString: endpoint, parameters: objectJSON)
            .responseArray { (dataResponse: DataResponse<[T]>) in
                self.processArrayResponse(persist: persist, dataResponse: dataResponse,
                                          completion: { (entities, error) -> () in
                                            completion(entities , error)
                })
        }
    }
    
    func postEntities<T:KliqueObject>(urlString: String, parameters: Parameters?,
                      completion:@escaping ([T],Error?) -> ()) {
        postEntities(persist: true, urlString: urlString, parameters: parameters, completion: completion)
    }
    
    func postEntities<T:KliqueObject>(persist: Bool, urlString: String, parameters: Parameters?,
                      completion:@escaping ([T],Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        
        
        self.createRequest(method: .post, URLString: endpoint,
                           parameters: parameters, encoding:JSONEncoding.default)
            .responseArray { (dataResponse : Alamofire.DataResponse<[T]>) -> Void in
                self.processArrayResponse(persist: persist, dataResponse: dataResponse,
                                          completion: { (entities, error) -> () in
                                            completion(entities , error)
                })
        }
    }
    
    func postEntity<T:KliqueObject>(urlString: String, object:KliqueObject, image: UIImage,
                    completion:@escaping (T?,Error?) -> ()) {
        postEntity(persist: true, urlString: urlString, object: object, image: image, completion: completion)
    }
    
    func postEntity<T:KliqueObject>(persist: Bool, urlString: String, object:KliqueObject, image: UIImage,
                    completion:@escaping (T?,Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        let objectJSON =  object.toJSONForCreate()
        debugPrint("objectJSON \(objectJSON)")
        self.createMultipartPOSTRequest(URLString: endpoint, parameters: objectJSON, image: image)
            .responseObject { (dataResponse : Alamofire.DataResponse<T>) -> Void in
                self.processObjectResponse(persist: persist, dataResponse: dataResponse,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func putEntity<T:KliqueObject>(urlString: String, object: KliqueObject,
                   completion:@escaping (T?,Error?) -> ()) {
        putEntity(persist: true, urlString: urlString, object: object, completion: completion)
    }
    
    func putEntity<T:KliqueObject>(persist: Bool, urlString: String, object: KliqueObject,
                   completion:@escaping (T?,Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        let objectJSON = object.toJSONForUpdate()
        
        self.createRequest(method: .put, URLString: endpoint, parameters: objectJSON)
            .responseObject { (response : Alamofire.DataResponse<T>) -> Void in
                self.processObjectResponse(persist: persist, dataResponse: response,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func putEntity<T:KliqueObject>(urlString: String, params: Parameters?,
                   completion:@escaping (T?,Error?) -> ()) {
        putEntity(persist: true, urlString: urlString, params: params, completion: completion)
    }
    
    func putEntity<T:KliqueObject>(persist: Bool, urlString: String, params: Parameters?,
                   completion:@escaping (T?,Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        
        self.createRequest(method: .put, URLString: endpoint, parameters: params)
            .responseObject { (response : Alamofire.DataResponse<T>) -> Void in
                self.processObjectResponse(persist: persist, dataResponse: response,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func deleteEntity<T:KliqueObject>(urlString: String, completion:@escaping (T?,Error?) -> ()) {
        deleteEntity(persist: true, urlString: urlString, completion: completion)
    }
    
    func deleteEntity<T:KliqueObject>(persist: Bool, urlString: String,
                      completion:@escaping (T?,Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        
        self.createRequest(method: .delete, URLString: endpoint, parameters: nil)
            .responseObject { (response : Alamofire.DataResponse<T>) -> Void in
                self.processObjectResponse(persist: false, dataResponse: response,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    func deleteEntities<T:KliqueObject>(urlString: String, completion:@escaping ([T],Error?) -> ()) {
        deleteEntities(persist: true, urlString: urlString, completion: completion)
    }
    
    func deleteEntities<T:KliqueObject>(persist: Bool, urlString: String,
                        completion:@escaping ([T],Error?) -> ()) {
        let endpoint: String = self.servicePath(endpoint: urlString)
        
        self.createRequest(method: .delete, URLString: endpoint, parameters: nil)
            .responseArray { (response : Alamofire.DataResponse<[T]>) -> Void in
                self.processArrayResponse(persist: persist, dataResponse: response,
                                          completion: { (entities, error) -> () in
                                            completion(entities , error)
                })
        }
    }
    
    func getEntity<T:KliqueObject>(urlString: String, parameters: Parameters?,
                   completion:@escaping (T?, Error?) -> ()) {
        let _ = getEntity(persist: true, urlString: urlString, parameters: parameters) { (entity, error) in
            completion(entity, error)
        }
    }
    
    func getEntity<T:KliqueObject>(persist: Bool, urlString: String, parameters: Parameters?,
                   completion:@escaping (T?, Error?) -> ()) -> Alamofire.DataRequest {
        
        let endpoint: String = self.servicePath(endpoint: urlString)
        return self.createRequest(method: .get, URLString: endpoint, parameters: parameters)
            .responseObject { (dataResponse : Alamofire.DataResponse<T>) -> Void in
                self.processObjectResponse(persist: persist, dataResponse: dataResponse,
                                           completion: { (entity, error) -> () in
                                            completion(entity , error)
                })
        }
    }
    
    func processObjectResponse<T:KliqueObject>(dataResponse: Alamofire.DataResponse<T>,
                               completion:(T?, Error?) -> ()) {
        processObjectResponse(persist: true, dataResponse: dataResponse, completion: completion)
    }
    
    func processObjectResponse<T:KliqueObject>(persist: Bool, dataResponse: Alamofire.DataResponse<T>,
                               completion:(T?, Error?) -> ()) {
        self.logResponseIssues(eventName: "API Request", dataResponse: dataResponse)
        
        if let data = dataResponse.data {
            let dataString = String(data: data, encoding:String.Encoding.utf8)
            //            debugPrint(dataString)
            var error  = self.errorForCode(response: dataResponse.response, data:dataString)
            let entity = dataResponse.result.value
            if error == nil && entity != nil && persist {
                error = self.saveEntity(entity: entity!)
            }
            completion(entity , error)
        }
        else {
            let error = NSError(domain: "klique.processObjectResponse", code: -1, userInfo: nil)
            completion(nil, error)
        }
    }
    
    func processArrayResponse<T:KliqueObject>(dataResponse: DataResponse<[T]>,
                              completion:([T], Error?) -> ()) {
        processArrayResponse(persist: true, dataResponse: dataResponse, completion: completion)
    }
    
    func processArrayResponse<T:KliqueObject>(persist: Bool, dataResponse: Alamofire.DataResponse<[T]>,
                              completion:([T], Error?) -> ()) {
        logResponseIssues(eventName: "API Request", dataResponse: dataResponse)
        
        if let data = dataResponse.data {
            let dataString = String(data: data, encoding:String.Encoding.utf8)
            //            debugPrint(dataString)
            var error = self.errorForCode(response: dataResponse.response, data: dataString)
            let entities = dataResponse.result.value ?? [T]()
            if error == nil && persist {
                error = self.saveEntities(entities: entities)
            }
            completion(entities, error)
        }
        else {
            let error = NSError(domain: "klique.processArrayResponse", code: -1, userInfo: nil)
            completion([T]() , error)
        }
    }
    
    
    func saveEntity<T:KliqueObject>(entity:T) -> NSError? {
        
        do {
            let realm = try Realm()
            try realm.write({ () -> Void in
                
                realm.add(entity, update: true)
            })
        }
        catch let error as NSError {
            
            return error
        }
        
        return nil
    }
    
    func saveEntities<T:KliqueObject>(entities:[T]) -> NSError? {
        
        do {
            let realm = try Realm()
            try realm.write({ () -> Void in
                
                realm.add(entities, update: true)
            })
        }
        catch let error as NSError {
            
            return error
        }
        
        return nil
    }
    
    func errorForCode(response : HTTPURLResponse?, data: String?) -> Error? {
        
        var error : NSError? = NSError(domain: "Unknown Error",
                                       code: response?.statusCode ?? 0,
                                       userInfo: nil)
        
        if let statusCode = response?.statusCode {
            switch statusCode {
                
            case 200..<400:
                
                error = nil
                
            case 401:
                
                error = NSError(domain: "Unauthorized", code: statusCode, userInfo: nil)
                
                if let object = data?.parseJSONString as? [String : Any] {
                    if let message = object["message"] as? String {
                        let userInfo = [NSLocalizedFailureReasonErrorKey: message]
                        error = NSError(domain: "Unauthorized", code: statusCode, userInfo: userInfo)
                    }
                }
                
            case 400:
                
                error = NSError(domain: "Bad Request", code: statusCode, userInfo: nil)
                
            case 404:
                
                error = NSError(domain: "Not Found", code: statusCode, userInfo: nil)
                
            default:
                error = NSError(domain: "Error", code: statusCode, userInfo: nil)
            }
            
        }
        return error
    }
    
    func createRequest(method: HTTPMethod, URLString: URLConvertible, parameters: Parameters? = nil,
                       headers: [String : String] = [:],
                       encoding: ParameterEncoding = URLEncoding.default) -> Alamofire.DataRequest {
        var finalHeaders = headers
        if let credential = type(of: self).credential {
            finalHeaders["Authorization"] = "Bearer \(credential.token)"
        } else {
            NSLog("Warning!  No credential set in KliqueService.")
        }
        
        var finalEncoding = encoding
        if method == .post || method == .put {
            
            finalEncoding = JSONEncoding.default
        }
        let request = Alamofire.request(URLString, method: method, parameters: parameters,
                                        encoding: finalEncoding, headers: finalHeaders)
            .response(completionHandler: { (dataResponse: DefaultDataResponse) in
                KliqueDebug.sharedInstance.resolveRequest()
                if dataResponse.response?.statusCode == 401 && type(of: self).credential != nil {
                    NotificationCenter.default.post(name: kNotificationTokenInvalidated, object: nil)
                }
            })
        
        KliqueDebug.sharedInstance.addRequest()
        debugPrint(request)
        return request
    }
    
    
    func createMultipartPOSTRequest(URLString: URLConvertible, parameters: Parameters? = [:],
                                    headers: [String : String] = [:],
                                    image: UIImage) -> Alamofire.DataRequest {
        
        var finalHeaders = headers
        let data = UIImageJPEGRepresentation(image, 1.0)!
        
        let multipartFormData = OMGMultipartFormData()
        multipartFormData.addFile(data, parameterName: "file",
                                  filename: "file.jpg",
                                  contentType: "image/jpg")
        if let parameters = parameters {
            multipartFormData.addParameters(parameters)
        }
        let request = try! OMGHTTPURLRQ.post(URLString.asURL().absoluteString, multipartFormData)
        let body = request.httpBody!
        
        finalHeaders = request.allHTTPHeaderFields!
        
        if let credential = type(of: self).credential {
            finalHeaders["Authorization"] = "Bearer \(credential.token)"
        }
        return Alamofire.upload(body, to: URLString, method: .post, headers: finalHeaders)
            .response(completionHandler: { (dataResponse: DefaultDataResponse) in
                if dataResponse.response?.statusCode == 401 && type(of: self).credential != nil {
                    NotificationCenter.default.post(name:kNotificationTokenInvalidated, object: nil)
                }
            })
    }
    
    // MARK: - Parameters Utils
    func dateStringFromDate(date:Date) -> String {
        
        let dateFormatter = DateFormatter()
        dateFormatter.dateFormat = KliqueService.dateFormatter
        dateFormatter.timeZone = TimeZone(secondsFromGMT: 0)
        return dateFormatter.string(from: date)
    }
    
    func logResponseIssues<T:AnyObject>(eventName: String, dataResponse: DataResponse<T>) {
        self.logResponseIssues(eventName: eventName, request: dataResponse.request,
                               error: dataResponse.error as? AFError,
                               resultNil: dataResponse.result.value == nil, timeline: dataResponse.timeline)
    }
    func logResponseIssues<T:AnyObject>(eventName: String, dataResponse: DataResponse<[T]>) {
        self.logResponseIssues(eventName: eventName, request: dataResponse.request,
                               error: dataResponse.error as? AFError,
                               resultNil: dataResponse.result.value == nil, timeline: dataResponse.timeline)
    }
    
    func logResponseIssues(eventName: String, request: URLRequest?, error: AFError?,
                           resultNil: Bool, timeline: Timeline) {
        if error != nil || resultNil {
            let userInfo: [String : Any] = [
                "url" : request!.url!.absoluteString,
                "resultNil" : resultNil
            ]
            //            if let error = error {
            //                userInfo["errorCode"] = error.code? ?? ""
            //                userInfo["errorDomain"] = error.domain? ?? ""
            //                userInfo["errorDescription"] = error.description? ?? ""
            //            }
            Answers.logCustomEvent(withName: "\(eventName) - Error", customAttributes: userInfo)
        }
        if timeline.totalDuration > KliqueService.slowDownloadThreshold {
            let userInfo: [String : Any] = [
                "url" : request?.url ?? "",
                "totalDuration": timeline.totalDuration,
                "requestDuration": timeline.requestDuration,
                "serializationDuration": timeline.serializationDuration
            ]
            Answers.logCustomEvent(withName: "\(eventName) - Slow", customAttributes: userInfo)
        }
    }
}
```

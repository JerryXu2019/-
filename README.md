# -
导航

import Foundation

@objc protocol YNAAppConfigUsableProtocol
{
    /**
     * アプリの時計
     */
    var appClock: YNAAppClockProtocol { get }

    /**
     * 有効期間内か確認
     */
    @objc optional func isValidTerm(_ startDateTime: Int, endDateTime: Int) -> Bool

    /**
     * 期間を過ぎているか確認
     */
    @objc optional func isExpiredTerm(_ endDateTime: Int) -> Bool

    /**
     * 対象バージョン範囲内か確認
     */
    @objc optional func isValidVersion(_ minVersion: Int, maxVersion: Int) -> Bool
}

extension YNAAppConfigUsableProtocol
{
    /**
     * 有効期間内か確認
     */
    func isValidTerm(_ startDateTime: Int, endDateTime: Int) -> Bool
    {
        // 有効期間中か否か（時刻による判定）
        var isValidTimeFromTime = false

        // 日時(unixtime)
        let nowDateTime = self.appClock.now().timeIntervalSince1970

        // 有効期間内か否か（時刻による判定）
        if startDateTime <= Int(nowDateTime) {
            // 表示開始日時が現在より前の場合

            if Int(nowDateTime) <= endDateTime {
                // 現在が表示終了日時より前の場合

                isValidTimeFromTime = true
            } else {
                isValidTimeFromTime = false
            }
        } else {
            isValidTimeFromTime = false
        }

        return isValidTimeFromTime
    }

    /**
     * 期間を過ぎているか確認
     */
    func isExpiredTerm(_ endDateTime: Int) -> Bool
    {
        // 期間を過ぎているか否か（時刻による判定）
        var isExpiredFromTime = false

        // 日時(unixtime)
        let dateTime = self.appClock.now().timeIntervalSince1970

        // 期間を過ぎているか否か（時刻による判定）
        if (endDateTime < Int(dateTime)) {
            isExpiredFromTime = true
        } else {
            isExpiredFromTime = false
        }

        return isExpiredFromTime;
    }

    /**
     * 対象バージョン範囲内か確認
     */
    func isValidVersion(_ minVersion: Int, maxVersion: Int = Int.max) -> Bool
    {

        // 対象バージョン範囲内か否か
        var isValidVersion = false

        let currentVersionStr = YNAPlatform.sharedInstance().appShortVersion
        let currentVersionArr = currentVersionStr.components(separatedBy: ".")
        var currentVersion = 0
        if currentVersionArr.count >= 3,
            let major = Int(currentVersionArr[0]),
            let minor = Int(currentVersionArr[1]),
            let build = Int(currentVersionArr[2]) {
            currentVersion = major * 10000 + minor * 100 + build
        }

        if (minVersion <= currentVersion) {
            // 最小バージョンが現バージョンより前の場合

            if (currentVersion <= maxVersion) {
                // 現バージョンが最大バージョンより前の場合

                isValidVersion = true
            } else {
                isValidVersion = false
            }
        } else {
            isValidVersion = false
        }
        
        return isValidVersion
    }
}

@objc protocol YNAAppClockProtocol
{
    /**
     * サーバ時間と端末時間から算出した現在時刻
     */
    func now() -> Date

    /**
     * 端末時間に不正があるか否かの判定
     */
    func isInvalidTime() -> Bool
}



import Foundation

public typealias DataTaskResult = (Data?, URLResponse?, Error?) -> Void

public protocol YNAURLSessionProtocol
{
    func dataTaskWithRequest(_ request: URLRequest, completionHandler: @escaping DataTaskResult) -> YNAURLSessionDataTaskProtocol
}

extension URLSession: YNAURLSessionProtocol
{
    public func dataTaskWithRequest(_ request: URLRequest, completionHandler: @escaping DataTaskResult) -> YNAURLSessionDataTaskProtocol
    {
        return (dataTask(with: request, completionHandler: completionHandler as (Data?, URLResponse?, Error?) -> Void) as URLSessionDataTask) as YNAURLSessionDataTaskProtocol
    }
}



import Foundation

protocol YNACampaignManagerProtocol {
    
    var countPeriodStartDate: Date? { get }
    var countPeriodEndDate: Date? { get }
    var drawingPeriodStartDate: Date? { get }
    var drawingPeriodEndDate: Date? { get }
    var countLocalPushInfo: [String: Any]? { get }
    var drawingLocalPushInfo: [String: Any]? { get }
    
    func requestGrantLot(_ completion: @escaping (_ count: Int, _ result: Bool) -> Void)
    func currentTrueDate() -> Date
}



import Foundation

enum HTTPMethod: String {
    case GET
    case POST
    case DELETE
    case PUT
}

/**
 認証リクエストプロトコル
 */
protocol YNAAuthorizedRequestProtocol {

    var url: String { get }
    var parameters: [String: String] { get }
    var method: HTTPMethod { get }
    
}

extension YNAAuthorizedRequestProtocol {

    /**
     認証リクエスト
     - note:
     - error.code によって処理を追加してください
     - YConnectErrorInvalidToken:
     アクセストークンの更新が失敗しているため､再度リクエストする処理を書いてください｡
     - YConnectErrorInvalidGrant:
     リフレッシュトークンの有効期限が切れているため､再度ログインする処理を書いてください｡
    */
    func request(_ completion: @escaping (_ error: NSError?, _ dictionary: [String: Any]?) -> Void) {

        guard let loginManager = YJLoginManager.sharedInstance() else {
            return
        }
        
        // accessToken更新処理
        if loginManager.isAccessTokenExpired() {

            loginManager.refreshAccessToken { _, error in

                if let error = error {
                    /// accessToken更新失敗
                    completion(error as NSError, nil)
                    return
                }

                /// accessToken更新成功したらリクエスト
                self.execute({ error, dictionary in
                    completion(error, dictionary)
                })
            }
            return
        }

        // リクエスト処理
        execute({ error, dictionary in
            completion(error, dictionary)
        })
    }


    fileprivate func execute(_ completion: @escaping (_ error: NSError?, _ dictionary: [String: Any]?) -> Void) {

        let loginManager = YJLoginManager.sharedInstance()
        let accessTokenString = loginManager?.accessTokenString()
        let client = YConnectAPIClient(accessToken: YConnectBearerToken(accessToken: accessTokenString, expiration: 0))

        /// リクエストパラメータ追加
        for (key, value) in parameters {
            client?.setParameter(key, value: value)
        }

        let handler: YConnectAPIClientResponseHandler = { response, _, error in

            if let error = error {
                completion(error as NSError, nil)
                return
            }

            do {
                guard let response = response,
                    let data = response.data(using: String.Encoding.utf8),
                    let dictionary = try JSONSerialization.jsonObject(with: data, options: .mutableContainers) as? [String: Any] else {
                    completion(nil, nil)
                    return
                }
                completion(error as? NSError, dictionary)
                
            } catch let error as NSError {
                completion(error, nil)
            }
        }

        client?.execute(url, method: method.rawValue, handler: handler)
    }
}


import Foundation

extension Timer
{
    /**
     NSTimer オブジェクトの作成

     - Parameters:
     - delay: タイマーを起動する前の遅延時刻
     - handler: タイマーが起動した時に呼び出されるブロック(クロージャ)

     - Returns: NSTimer オブジェクト
     */
    class func schedule(delay: TimeInterval, handler: @escaping (Timer?) -> Void) -> Timer?
    {
        let fireDate = delay + CFAbsoluteTimeGetCurrent()

        // Parameters:
        //      kCFAllocatorDefault: 新しいオブジェクト用にメモリの割り当て
        //      fireDate: タイマーが最初に起動する時刻
        //      0: タイマーの起動間隔
        //      0: 無効
        //      0: 無効
        //      handler: タイマーが起動した時に呼び出されるブロック
        let timer = CFRunLoopTimerCreateWithHandler(kCFAllocatorDefault,
                                                    fireDate,
                                                    0, 0, 0,
                                                    handler)
        CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, CFRunLoopMode.commonModes)
        return timer
    }

    /**
     NSTimer オブジェクトの作成

     - Parameters:
     - repeatInterval: タイマーの起動間隔
     - handler: タイマーが起動した時に呼び出されるブロック(クロージャ)

     - Returns: NSTimer オブジェクト
     */
    class func schedule(repeatInterval interval: TimeInterval, handler: @escaping (Timer?) -> Void) -> Timer?
    {
        let fireDate = interval + CFAbsoluteTimeGetCurrent()

        // Parameters:
        //      kCFAllocatorDefault: 新しいオブジェクト用にメモリの割り当て
        //      fireDate: タイマーが最初に起動する時刻
        //      interval: タイマーの起動間隔
        //      0: 無効
        //      0: 無効
        //      handler: タイマーが起動した時に呼び出されるブロック
        let timer = CFRunLoopTimerCreateWithHandler(kCFAllocatorDefault,
                                                    fireDate,
                                                    interval,
                                                    0, 0,
                                                    handler)
        CFRunLoopAddTimer(CFRunLoopGetCurrent(), timer, CFRunLoopMode.commonModes)
        return timer
    }
}

extension UIView {
    /**
     * viewを点滅させる
     *
     * @param withDuration 点滅間隔（秒）
     * @param delay 点滅開始を遅らせる間隔（秒）
     * @param repeatCount 点滅させる回数
     * @param after 処理終了ハンドラ
     */
    func flash(withDuration: TimeInterval, delay: TimeInterval = 0.0, repeatCount: Float = 1, after:((Bool) -> Void)? = nil) {
        self.alpha = 0.4
        UIView.animate(withDuration: withDuration, delay: delay, options: .allowUserInteraction, animations: {
            UIView.setAnimationRepeatCount(repeatCount)
            self.alpha = 1.0
        }, completion: { [weak self] isCompleted in
            self?.alpha = 1.0
            after?(isCompleted)
        })
    }
}


extension UIImage {
    
    /**
     * グラデーションimageを取得
     */
    func getChangeAlpha(alpha: CGFloat) -> UIImage? {
        UIGraphicsBeginImageContextWithOptions(size, false, scale)
        draw(at: .zero, blendMode: .normal, alpha: alpha)
        let newImage = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()
        return newImage
    }
    
    /**
     * リサイズ imageを取得
     */
    func resize(width: CGFloat, height: CGFloat) -> UIImage? {
        let size = CGSize(width: width, height: height);
        UIGraphicsBeginImageContextWithOptions(size, false, 0);
        self.draw(in: CGRect(x: 0, y: 0, width: size.width, height: size.height));
        
        let image = UIGraphicsGetImageFromCurrentImageContext();
        UIGraphicsEndImageContext();
        
        return image;
    }
    
    /**
     * Clear imageを取得 (画像用)
     */
    class func getClearImage(width: CGFloat) -> UIImage? {
        
        let size = CGSize(width: width, height: 1);
        
        UIGraphicsBeginImageContextWithOptions(size, false, 0)
        guard let context = UIGraphicsGetCurrentContext() else {
            return nil
        }
        context.setFillColor(UIColor.clear.cgColor)
        let rect = CGRect(origin: .zero, size: size)
        context.fill(rect)
        
        // 画像に変換する
        let image = UIGraphicsGetImageFromCurrentImageContext()
        UIGraphicsEndImageContext()

        return image;
    }
}


import Foundation

final class UserAgent: NSObject, UserAgentProtocol {
    func value() -> String {
        let version = YNAPlatform.sharedInstance().appShortVersion
        let bundleId = YNAPlatform.sharedInstance().appId
        let osIdentifier = "YJApp-IOS"

        return osIdentifier + " " + bundleId + "/" + version
    }
}


import Foundation

final class YNAParkingPlaceData: NSObject, NSCoding
{
    /**
     * 駐車した時刻(UNIXタイムスタンプ)
     */
    var time:Date?
    /**
     * 駐車した位置(緯度)
     */
    var latitude = 0.0
    /**
     * 駐車した位置(経度)
     */
    var longitude = 0.0

    override init() {
        super.init()
        self.time = Date()
    }

    init(latitude: Double, longitude: Double) {
        super.init()
        self.time = Date()
        self.latitude = latitude
        self.longitude = longitude
    }

    func encode(with aCoder: NSCoder) {
        aCoder.encode(time, forKey: "time")
        aCoder.encode(latitude, forKey: "latitude")
        aCoder.encode(longitude, forKey: "longitude")
    }

    required init?(coder aDecoder: NSCoder) {
        time = aDecoder.decodeObject(forKey: "time") as? Date
        latitude = aDecoder.decodeDouble(forKey: "latitude")
        longitude = aDecoder.decodeDouble(forKey: "longitude")
    }
}

import UIKit

@objc class YNAOverlaidUIView: UIView {
    
    private static let kFontColor = UIColor.white
    private static let kTextFontSize: CGFloat = 28
    static let kDefaultColor = UIColor(displayP3Red: 28/255, green: 70/255, blue: 197/255, alpha: 1)
    static let kHighwayColor = UIColor(displayP3Red: 2/255, green: 136/255, blue: 73/255, alpha: 1)
    static let kStartGuideTitle = "運転を開始すると\n案内が始まります"
    static let kEndGuideTitle = "目的地に到着しました\n案内を終了します"
    static let kUpdateGuideTitle = "危険なため運転中は\nスマートフォンを\n触らないでください"
    
    init() {
        super.init(frame: CGRect.zero)
        
        self.autoresizingMask = [.flexibleWidth, .flexibleHeight]
        backgroundColor = YNAOverlaidUIView.kDefaultColor
        
        let label = UILabel()
        
        label.backgroundColor = UIColor.clear
        label.textColor = YNAOverlaidUIView.kFontColor
        label.font = UIFont.boldSystemFont(ofSize: YNAOverlaidUIView.kTextFontSize)
        label.lineBreakMode  = .byTruncatingHead
        label.numberOfLines  = 0;
        label.textAlignment = .center
        
        self.addSubview(label)
    }
    
    required init?(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)
    }
    
    func setText(_ text:String) {
        guard let label = self.label() else {
            return
        }
        label.text = text
    }
    
    func label() -> UILabel? {
        for subView in self.subviews {
            if let itemLabel = subView as? UILabel {
                return itemLabel
            }
        }
        return nil
    }
    
    override func layoutSubviews() {
        super.layoutSubviews()
        
        guard let label = self.label() else {
            return
        }
        let safeAreaInsets = YNAHelper.getSafeAreaInsets()
        let marginX = safeAreaInsets.left
        let marginY = safeAreaInsets.bottom + CGFloat(CONST_ROUTE_GUIDE_DEF_ARRIVALVIEW_HEIGHT_LAND)
        frame = CGRect(x: 0,
                       y: 0,
                       width: UIScreen.main.bounds.width + marginX,
                       height: UIScreen.main.bounds.height - marginY)
        label.frame = frame
    }
}

import UIKit

final class YNAJtisTimeView: UIView {

    @IBOutlet weak var timeLabel: UILabel!
    
    init() {
        super.init(frame: CGRect.zero)
        loadXib()
    }
    
    required init(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)!
        loadXib()
    }
    
    private func loadXib() {
        // xibファイル読み込み
        let bundle = Bundle(for: type(of: self))
        let nib = UINib(nibName: "YNAJtisTimeView", bundle: bundle)
        let view = nib.instantiate(withOwner: self, options: nil).first as! UIView
        
        self.bounds = view.frame
        
        addSubview(view)
    }
}


import UIKit

protocol YNAParkingPointViewDelegate: class {
    func didTapCamera()
    func didTapNavi()
}

class YNAParkingPointView: UIView {

    @IBOutlet weak var parkingTimeLabel: UILabel!

    weak var delegate: YNAParkingPointViewDelegate?

    /// 保存された日時
    fileprivate var saveDate: Date = Date()

    /// 周期的な処理を行うためのタイマーインスタンス
    private(set) var timer: Timer = Timer()

    override init(frame: CGRect) {
        super.init(frame: frame)
        loadXib()
    }

    required init(coder aDecoder: NSCoder) {
        super.init(coder: aDecoder)!
        loadXib()
    }

    override func willMove(toSuperview newSuperview: UIView?) {
        super.willMove(toSuperview: newSuperview)
        self.timer.invalidate()
    }

    private func loadXib() {
        // xibファイル読み込み
        let bundle = Bundle(for: type(of: self))
        let nib = UINib(nibName: "YNAParkingPointView", bundle: bundle)
        let view = nib.instantiate(withOwner: self, options: nil).first as! UIView

        view.frame = self.bounds

        addSubview(view)
    }

    @IBAction func tapCameraButton(_ sender: Any) {
        delegate?.didTapCamera()
    }

    @IBAction func tapNaviButton(_ sender: Any) {
        delegate?.didTapNavi()
    }

    /// 駐車位置の保存時間を表示する
    ///
    /// - Parameter saveDate: 保存時間
    func displayParkingTime(saveDate: Date) {
        self.saveDate = saveDate

        // 2重起動させないよう動いてしまっているタイマーがあればストップ
        if timer.isValid {
            timer.invalidate()
        }
        // 1秒ごとに処理(calculateParkingTime)を実行
        timer = Timer.scheduledTimer(timeInterval: 1,
                                     target: self,
                                     selector: #selector(self.calculateParkingTime),
                                     userInfo: nil,
                                     repeats: true)
    }

    /// 現在時間と保存時間の差分から駐車時間を計算する
    @objc fileprivate func calculateParkingTime() {
        // 現在時間と保存時間の差分
        let diff = Date().timeIntervalSince(saveDate)
        guard diff >= 0 else {
            parkingTimeLabel.text = "--時間--分"
            return
        }

        let sec = Double(diff)
        var min = Int(ceil(sec / 60))
        let hour = min / 60

        if hour >= 72 {
            parkingTimeLabel.text = "3日以上"
        } else {
            if min / 60 >= 1 {
                min = min % 60
            }
            parkingTimeLabel.text = "\(String(format: "%d", hour))時間 \(String(format: "%02d", min))分"
        }
    }
}


import Foundation
import Social

@objc protocol YNAAnalyzeDriveResultSnsButtonDelegate
{
    func analyzeDriveResultSnsButtonDidSelect(_ view: YNAAnalyzeDriveResultSnsButton,
                                              type: String)
}

final class YNAAnalyzeDriveResultSnsButton: UIView {

    static let kInnerButtonMargin : CGFloat = 8

    // ボタン
    let facebookShareButton = YNAFeedbackButton()
    let twitterShareButton = YNAFeedbackButton()
    let otherShareButton = YNAFeedbackButton()

    var enabled: Bool = true {
        didSet {

            if oldValue != self.enabled {
                // 値が変更した場合

                self.facebookShareButton.isEnabled = self.enabled;
                self.twitterShareButton.isEnabled = self.enabled;
                self.otherShareButton.isEnabled = self.enabled;
            }
        }
    }

    weak var delegate: YNAAnalyzeDriveResultSnsButtonDelegate? // デリゲート

    init()
    {
        super.init(frame: CGRect.zero)

        // Facebook
        self.facebookShareButton.backgroundColor = UIColor.clear
        self.facebookShareButton.layer.cornerRadius = 4.0
        self.facebookShareButton.clipsToBounds = true
        self.facebookShareButton.isEnabled = self.enabled
        self.facebookShareButton.addTarget(self,
                                           action:#selector(selectFacebookShareButtonTouchUpInside(_:)),
                                           for: .touchUpInside)
        self.facebookShareButton.setImage(UIImage(named: "common_share_fb"),
                                          for: UIControlState())
        self.facebookShareButton.setImage(UIImage(named: "common_share_fb_disable"),
                                          for: .disabled)

        // Twitter
        self.twitterShareButton.backgroundColor = UIColor.clear
        self.twitterShareButton.layer.cornerRadius = 4.0
        self.twitterShareButton.clipsToBounds = true
        self.twitterShareButton.isEnabled = self.enabled
        self.twitterShareButton.addTarget(self,
                                          action:#selector(selectTwitterShareButtonTouchUpInside(_:)),
                                          for: .touchUpInside)
        self.twitterShareButton.setImage(UIImage(named: "common_share_tw"),
                                         for: UIControlState())
        self.twitterShareButton.setImage(UIImage(named: "common_share_tw_disable"),
                                          for: .disabled)

        // Other
        self.otherShareButton.backgroundColor = UIColor.clear
        self.otherShareButton.layer.cornerRadius = 4.0
        self.otherShareButton.clipsToBounds = true
        self.otherShareButton.isEnabled = self.enabled
        self.otherShareButton.addTarget(self,
                                        action:#selector(selectOtherShareButtonTouchUpInside(_:)),
                                        for: .touchUpInside)
        self.otherShareButton.setImage(UIImage(named: "common_share_green"),
                                       for: UIControlState())
        self.otherShareButton.setImage(UIImage(named: "common_share_green_disable"),
                                       for: .disabled)

        self.facebookShareButton.sizeToFit()
        self.twitterShareButton.sizeToFit()
        self.otherShareButton.sizeToFit()

        //iOS11以上はツイートボタンを非表示にする
        if #available(iOS 11, *) {
            self.twitterShareButton.isHidden = true
            self.twitterShareButton.frame.size.width = 16
        }

        self.addSubview(self.facebookShareButton)
        self.addSubview(self.twitterShareButton)
        self.addSubview(self.otherShareButton)
    }

    required init?(coder aDecoder: NSCoder)
    {
        super.init(coder: aDecoder)
    }

    override func layoutSubviews()
    {
        super.layoutSubviews()

        // ボタンの配置
        self.layoutButtons()
    }

    fileprivate func layoutButtons()
    {
        var fbShareBtnFrame = self.facebookShareButton.frame
        var twShareBtnFrame = self.twitterShareButton.frame
        var otShareBtnFrame = self.otherShareButton.frame

        fbShareBtnFrame.origin.x = self.bounds.size.width / 2
            - twShareBtnFrame.size.width / 2
            - YNAAnalyzeDriveResultSnsButton.kInnerButtonMargin
            - fbShareBtnFrame.size.width
        fbShareBtnFrame.origin.y = 0
        self.facebookShareButton.frame = fbShareBtnFrame

        twShareBtnFrame.origin.x = self.bounds.size.width / 2
            - twShareBtnFrame.size.width / 2
        twShareBtnFrame.origin.y = 0
        self.twitterShareButton.frame = twShareBtnFrame

        otShareBtnFrame.origin.x = self.bounds.size.width / 2
            + twShareBtnFrame.size.width / 2
            + YNAAnalyzeDriveResultSnsButton.kInnerButtonMargin
        otShareBtnFrame.origin.y = 0
        self.otherShareButton.frame = otShareBtnFrame
    }

    func selectFacebookShareButtonTouchUpInside(_ sender: UIButton)
    {
        guard let delegate = self.delegate else {
            return
        }
        delegate.analyzeDriveResultSnsButtonDidSelect(self, type: SLServiceTypeFacebook)
    }

    func selectTwitterShareButtonTouchUpInside(_ sender: UIButton)
    {
        guard let delegate = self.delegate else {
            return
        }
        delegate.analyzeDriveResultSnsButtonDidSelect(self, type: SLServiceTypeTwitter)
    }

    func selectOtherShareButtonTouchUpInside(_ sender: UIButton)
    {
        guard let delegate = self.delegate else {
            return
        }
        delegate.analyzeDriveResultSnsButtonDidSelect(self, type: "")
    }
}

import XCTest
@testable import YNaviApp

class YNAAnalyzeDriveRankingDataTests: XCTestCase {

    override func setUp() {
        super.setUp()
        // Put setup code here. This method is called before the invocation of each test method in the class.
    }

    override func tearDown() {
        // Put teardown code here. This method is called after the invocation of each test method in the class.
        super.tearDown()
    }

    func testInitWithInvalidObject() {
        let data = YNAAnalyzeDriveRankingData(jsonObject: nil)
        XCTAssertNotNil(data)
        XCTAssertEqual(data.week, YNAAnalyzeDriveRankingData().week)
        XCTAssertEqual(data.startDate, YNAAnalyzeDriveRankingData().startDate)
        XCTAssertEqual(data.endDate, YNAAnalyzeDriveRankingData().endDate)
        XCTAssertEqual(data.ranks.count, YNAAnalyzeDriveRankingData().ranks.count) // 0
    }

    func testInitWithEmptyObject() {
        let week = 1473033600
        let start = "2016/9/5"
        let end = "2016/9/11"
        let resultsItemJson: [String: Any] = [
            "Week": week,
            "Start": start,
            "End": end,
            "Ranks":[],
        ]

        let data = YNAAnalyzeDriveRankingData(jsonObject: resultsItemJson)

        XCTAssertEqual(data.week, week)
        XCTAssertEqual(data.startDate, start)
        XCTAssertEqual(data.endDate, end)
        XCTAssertEqual(data.ranks.count, 0)
    }

    func testInitWithValidObject() {
        let week = 1473033600
        let start = "2016/9/5"
        let end = "2016/9/11"
        let resultsItemJson: [String: Any] = [
            "Week": week,
            "Start": start,
            "End": end,
            "Ranks": [
                [
                    "PrefCode": "13",
                    "PrefName": "東京都",
                    "Score": 88.88,
                    "Count": 10,
                    "Rank": 987654321,
                    "Mileage": 8888.8,
                    "UpdateTimestamp": 1473638400,
                    "UpdateTimeName": "9/12 00:00",
                ],
                [
                    "PrefCode": "14",
                    "PrefName": "神奈川県",
                    "Score": 0.88,
                    "Count": 1,
                    "Rank": 123456789,
                    "Mileage": 10.1,
                    "UpdateTimestamp": 1473638400,
                    "UpdateTimeName": "9/12 00:00",
                ],
            ],
        ]

        let data = YNAAnalyzeDriveRankingData(jsonObject: resultsItemJson)

        XCTAssertEqual(data.week, week)
        XCTAssertEqual(data.startDate, start)
        XCTAssertEqual(data.endDate, end)
        XCTAssertEqual(data.ranks.count, 2)

        guard let ranks = resultsItemJson["Ranks"] as? [[String: Any]] else {
            XCTFail("ランキングデータありの json には Ranks が存在するはず")
            return
        }

        for (index, rank) in ranks.enumerated() {
            XCTAssertTrue(data.ranks[index] is YNAAnalyzeDriveRankingRankData)
            XCTAssertEqual(data.ranks[index].prefCode, rank["PrefCode"] as? String)
            XCTAssertEqual(data.ranks[index].prefName, rank["PrefName"] as? String)
            XCTAssertEqual(data.ranks[index].score, rank["Score"] as? Double)
            XCTAssertEqual(data.ranks[index].count, rank["Count"] as? Int)
            XCTAssertEqual(data.ranks[index].rank, rank["Rank"] as? Int)
            XCTAssertEqual(data.ranks[index].previousRank, YNAAnalyzeDriveRankingRankData().previousRank, "previousRank はマネージャークラスで代入する")
            XCTAssertEqual(data.ranks[index].mileage, rank["Mileage"] as? Double)
            XCTAssertEqual(data.ranks[index].updateTimestamp, rank["UpdateTimestamp"] as? Int)
            XCTAssertEqual(data.ranks[index].updateTimeName, rank["UpdateTimeName"] as? String)
        }
    }

}
